# SimpleLogin Inbound Bounce Handling — Enumeration-Oracle Investigation

> **Evidence-based security investigation.** Every behavioral claim below is backed either by the **actual, verbatim SMTP status string** captured from a run of the canonical inbound handler (with the command and handler layer that produced it), or by a `file:line` citation into the source. Statements that are reasoned rather than directly observed are explicitly labeled **(inferred)**. Complete, verbatim transcripts and the full source of every temporary observation script are embedded in **Phase K (Evidence Appendix)**. The source repository was left byte-for-byte unchanged apart from this one document; all temporary observation scripts lived **outside** the repository checkout (container path `/tmp/blitzy_evidence/`, host backup `/tmp/blitzy_evidence_host/`) and were removed afterward (see **Phase J**).

---

## Direct Answer (read this first)

**YES — SimpleLogin's inbound bounce path exposes a recipient/identifier enumeration oracle on the OLDER, unsigned VERP address format.** An external sender who controls only the SMTP envelope can distinguish a *valid* `email_log_id` from an *invalid* one purely by reading the SMTP status string. Probed with an ordinary, non-bounce message through the real inbound handler chain, the two ids diverge as follows (verbatim, captured via `handle_DATA()`):

- **invalid id** `bounce+99999999999999+@sl.local` → **`550 SL E512 No such email log`**
- **valid id** `bounce+385+@sl.local` → **`250 SL E213 Unknown email ignored`**

The two responses differ at **both** the numeric SMTP level (`550` vs `250`) **and** the SimpleLogin sub-code (`E512` vs `E213`).

The oracle is **richer than mere existence**: a valid id whose owning user is *inactive* produces a third, distinct response — **`550 SL E510 so such user`** (the source's literal typo is preserved) — giving a three-way discrimination between *nonexistent* (`E512`), *existent-but-inactive* (`E510`), and *existent-active* (`E211` for a real bounce / `E213` for a non-bounce probe).

The **NEWER, cryptographically-SIGNED** VERP format closes this gap. Its payload is **not encrypted** — the `[verp_type, object_id, minutes-since-epoch]` list is base32-encoded **cleartext** — but it is **authenticated** by a truncated HMAC (`sha3-224`, 8 bytes) computed by `generate_verp_email()` [app/email_utils.py:L1438-L1464] and verified by `get_verp_info_from_email()` [app/email_utils.py:L1467-L1498]. A forged (bad-HMAC) or far-future (expired) signed address causes `get_verp_info_from_email()` to return `None`, which diverts the message away from the email-log lookup entirely — it falls through to normal alias handling and returns **`550 SL E515 Email not exist`**, never reaching the `E512`/`E213` oracle. The unsigned path has **no** such cryptographic gate.

**One nuance reported exactly as observed (not adjusted):** the expiry check in `get_verp_info_from_email()` is **one-sided** — it rejects only timestamps too far in the *future* [app/email_utils.py:L1496]; there is **no lower bound**, so an *old* signed address (observed at `-6` days) is **still accepted** and behaves like a good signature. This is the opposite of a short-lived replay-limiting tag and is reported plainly in Phase F.

**A second nuance:** when the receiving MTA's SPF verdict for the probe's return-path is *fail* or *soft-fail*, a **returned** `5XX` (such as `E512` or `E510`) is rewritten to **`250 SL E216 Handled spf policy`**, which makes the *numeric* code converge with the valid-id case. **But (a)** the SimpleLogin sub-code string still differs (`E216` vs `E213`), so the oracle survives at the string level; and **(b)** the valid-id non-bounce branch **raises an exception before the rewrite runs**, so it is never suppressed — under SPF-fail the invalid id yields `E216` while the valid id yields `E213` (still two distinct strings). This rewrite is driven by the receiver's rspamd header, **not** by anything the external sender sets on the SMTP envelope.

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
| 5 | The exact SMTP responses for valid vs invalid vs non-bounce | Phase D (+ Phase K appendix) |
| 6 | Bounce detection (`is_bounce`) and its spoofability | Phase E |
| 7 | Oracle richness (does it reveal more than existence?) and the ordering that makes the leak observable | Phase D |

### A.2 Methodology — run-first, canonical path only

All responses were captured by driving the **real inbound chain** used for every message SimpleLogin receives:

```
MailHandler.handle_DATA()   [email_handler.py:L2289]   (async aiosmtpd DATA callback)
      -> _handle()          [email_handler.py:L2335]   (applies the SPF 5XX->E216 rewrite)
          -> handle()       [email_handler.py:L1945]   (the routing dispatcher)
```

No debug hooks and no hand-written signed addresses were used as a source of any observed value, and **no observed value (no SMTP status string, no state transition) was produced under any mock**: the verifier `get_verp_info_from_email()`, the routing `handle()`, the `_handle()` SPF layer, the `handle_DATA()` entry point, the database, and the final SMTP response path all ran **unmocked** for every captured value. Signed probes were produced by the software's **own** signer, `generate_verp_email()` [app/email_utils.py:L1438-L1464]. The **one** use of `unittest.mock.patch` anywhere in the observation scripts is a **signer-clock shim** in Script C (Phase K.4): to construct the two timestamp-boundary *input* addresses (FUTURE `+6d` and OLD `-6d`), `patch("app.email_utils.time")` temporarily shifts the clock the signer reads **while generating those two addresses only**. It patches nothing in the verifier or the handler chain, and it is released before the address is submitted; the resulting addresses are ordinary inputs whose *observed* outcomes (`E515` for the expired one, `E213` for the still-valid one) are produced by the real, unpatched verifier and handler. See the clarifying note in Phase K.4. Fixtures (a real `User`, `Alias`, `Contact`, `EmailLog`) were minted through the project's own models against a **real PostgreSQL database**, using the project's own bootstrap (`create_app()`, `add_sl_domains()`, `add_proton_partner()` — the same calls `tests/conftest.py` makes) and helpers (`tests/utils.py::create_new_user` [tests/utils.py:L17]), so the runtime is identical to how the project itself exercises these paths.

Because the wire response is composed across **three layers in three different methods**, every matrix cell was driven through each relevant layer and the response captured at each:

1. `handle()` [email_handler.py:L1945] — returns a status string **or raises** a `VERP*` exception (the raw routing outcome).
2. `_handle()` [email_handler.py:L2335] — may rewrite a returned `5XX` to `E216` based on the SPF verdict.
3. `handle_DATA()` [email_handler.py:L2289] — maps a raised `VERP*` exception to `E213` and any other exception to `E404`.

Observing only one layer would miss either the SPF suppression or the exception mapping, and would misreport the oracle's observability. Where a scenario is exercised by calling a helper (e.g. `parse_id_from_bounce`, `is_bounce`, `get_verp_info_from_email`) directly rather than through `handle_DATA`/`_handle`/`handle`, that call is explicitly labeled **(non-canonical, direct call)** at the point of use.

### A.3 Canonical environment (actual, with the mandated-vs-observed discrepancy disclosed)

The exact runtime facts were captured with the commands shown; the complete transcript is embedded in **Phase K.1**.

| Fact | Command | Observed value |
|------|---------|----------------|
| Python | `python --version` | **3.10.18** |
| aiosmtpd | `python -c "import aiosmtpd; print(aiosmtpd.__version__)"` | **1.4.2** |
| Database engine | `SELECT version();` | in-image: **PostgreSQL 15.13 (Debian 15.13-0+deb12u1)** on port `15432`; additionally provisioned to satisfy the AAP-mandated major version: **PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1)** — the complete canonical matrix was re-run against **both**, byte-identical (Phase **K.1a**) |
| Redis | `redis-server --version` | **7.0.15** |
| Migration head | `SELECT version_num FROM alembic_version;` | **`32f25cbf12f6`** |
| Schema size | `SELECT count(*) FROM information_schema.tables WHERE table_schema='public';` | **77** tables |
| Config | `CONFIG=/work/tests/test.env` | `EMAIL_DOMAIN=sl.local`, `NOT_SEND_EMAIL=true`, `pg_trgm` enabled |

**Discrepancy disclosure (reported exactly, not normalized):** the Agent Action Plan and its Section 0.4.1 dependency table specify **PostgreSQL 13** and `aiosmtpd 1.4.6`. The canonical Docker image ships **PostgreSQL 15.13** and **`aiosmtpd 1.4.2`** (the version actually pinned in `poetry.lock`) as its in-image runtime. To remove any doubt that the mandated datastore matters, the AAP-mandated **PostgreSQL 13** was additionally provisioned (**13.23**) and the **entire canonical `handle_DATA` matrix re-run against it, twice**; every wire response is **byte-identical** to the PostgreSQL 15 baseline (complete provisioning commands, both full transcripts, and an executable cross-engine comparison are embedded in **Phase K.1a**). The `aiosmtpd` version difference (1.4.6 mandated vs 1.4.2 actual) is likewise reported as the *actual* runtime. Neither engine nor library version affects any bounce-routing response: the routing and every status string are implemented entirely inside `email_handler.py` and `app/email/status.py`, which are database- and aiosmtpd-version-independent; the PostgreSQL major version only affects where the real `EmailLog`/`Alias`/`User` rows are stored, which is exactly what makes the valid-vs-invalid id distinction meaningful — and the PostgreSQL 13 run confirms the responses are the same on the mandated engine.

**Invocation form for every observation cell** (the scripts live in the container's `/tmp/blitzy_evidence/` — a non-bind-mounted overlay — and were mirrored to the host at `/tmp/blitzy_evidence_host/`, both **outside** the repository checkout):

```
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/<script>.py
```

**(inferred / environmental note):** `re2` bindings are provided by the canonical compiled `pyre2` 0.3.6 (matching `poetry.lock`); `re2` is used only for unrelated alias-regex matching in `app/spamassassin_utils.py` and is **off the bounce-routing path**, so it cannot affect any status string reported here.

### A.4 The three existing SPF tests — described narrowly, with transcript

The repository ships three tests that touch this area: `test_prevent_5xx_from_spf`, `test_preserve_5xx_with_valid_spf`, and `test_preserve_5xx_with_no_header` [tests/test_email_handler.py]. **They are narrow:** each constructs an envelope whose single recipient is `generate_verp_email(VerpType.bounce_forward, 99999999999999)` (an **invalid signed** id) and calls **`MailHandler()._handle(envelope, msg)` only** — i.e. they exercise the `_handle()` SPF-rewrite layer for one invalid-id cell (fail->`E216`, allow->`E512`, no-header->`E512`). They do **not** exercise `handle_DATA()`, the valid-id branch, the reply/transactional/iCloud branches, `is_bounce`, or the unsigned format. They are cited here only as corroboration that the `_handle()` SPF-rewrite path behaves as this investigation independently observed, and as confirmation that the harness driving these observations is the canonical one. They **pass** in this environment:

```
$ python -m pytest tests/test_email_handler.py -k "prevent_5xx or preserve_5xx" -v --no-header -p no:cacheprovider
tests/test_email_handler.py::test_prevent_5xx_from_spf PASSED            [ 33%]
tests/test_email_handler.py::test_preserve_5xx_with_valid_spf PASSED     [ 66%]
tests/test_email_handler.py::test_preserve_5xx_with_no_header PASSED     [100%]
================ 3 passed, 20 deselected, 18 warnings in 0.98s =================
```

(The complete pytest transcript, including the 18 deprecation warnings, is embedded in **Phase K.1**.)

### A.5 Database-state reality (reported exactly as observed — important for the state sections)

The mutating scenarios (Phase D.3, Phase F) **commit** rows to the database. In this environment — **SQLAlchemy 1.3.24**, an app-level `scoped_session` bound to a single `engine.connect()` connection [app/db.py:L9-L14] — a `Session.commit()` is **not** rolled back by the harness. This was verified empirically: neither a standalone outer `connection.begin()/transaction.rollback()` nor the project's own `flask_client` fixture reverts committed rows in this container. Consequently, state transitions below are reported honestly as **committed mutations to the ephemeral, in-container test database** (row counts advance run-to-run), **not** as transient changes that are rolled back. This does **not** affect the source repository, which is verified byte-for-byte unchanged (Phase J); the test database is disposable container state. Each mutating scenario mints **fresh** fixtures, so the observed *deltas* are stable even though absolute row ids grow across runs.

### A.6 Determinism

Every matrix cell was executed **at least twice** and was **stable** (identical outcome sequence across runs, after normalizing volatile tokens such as random fixture names, per-run message-ids, and the time-bucketed base32 signature). As a dedicated determinism spot-check, the **same unchanged inputs** were driven repeatedly:

- unsigned invalid-id probe `bounce+99999999999999+@sl.local` through `_handle()` — **5×** -> `550 SL E512 No such email log` (5/5 identical)
- signed invalid-id probe through `_handle()` — **5×** -> `550 SL E512 No such email log` (5/5 identical)
- unsigned valid-id probe `bounce+402+@sl.local` (non-bounce) through `handle_DATA()` — **5×** -> `250 SL E213 Unknown email ignored` (5/5 identical)

The complete five-run transcript for each is embedded in **Phase K.9** (script `obs_G_determinism.py`, transcript `obs_G.out`).

---

## Phase B — Objective 1: The two address formats

SimpleLogin accepts **two** shapes of bounce address on inbound mail. They coexist because the older format predates the signed one and is retained for backward compatibility.

### B.1 OLD, human-readable, UNSIGNED VERP

Shape: **`bounce+{email_log_id}+@{EMAIL_DOMAIN}`**, assembled from two config constants [app/config.py:L100-L101]:

```python
BOUNCE_PREFIX = os.environ.get("BOUNCE_PREFIX") or "bounce+"
BOUNCE_SUFFIX = os.environ.get("BOUNCE_SUFFIX") or f"+@{EMAIL_DOMAIN}"
```

Note that `BOUNCE_SUFFIX` **already includes** the domain (`+@sl.local`), so the full unsigned address is `BOUNCE_PREFIX + str(id) + BOUNCE_SUFFIX` = `bounce+{id}+@sl.local` — the domain is not appended a second time. In the test domain (`EMAIL_DOMAIN=sl.local`) this is `bounce+{id}+@sl.local`. The id is recovered by a **plain integer parse with no signature and no verification whatsoever** — `parse_id_from_bounce()` [app/email_utils.py:L1258-L1259], quoted complete:

```python
def parse_id_from_bounce(email_address: str) -> int:
    return int(email_address[email_address.find("+") : email_address.rfind("+")])
```

**Observed (non-canonical, direct call)** — this reproduces the user's example `bounce+12345+@domain` verbatim in the test domain (full transcript: Phase K.2, `obs_A.out`):

```
parse_id_from_bounce('bounce+12345+@sl.local') = 12345
parse_id_from_bounce('bounce+369+@sl.local') = 369
parse_id_from_bounce('bounce+99999999999999+@sl.local') = 99999999999999
```

The integer `12345` is exactly the argument later passed to `EmailLog.get(id)` (Phase C), so this address *is* the enumeration probe: substitute a valid vs. an invalid integer and the id is fed straight into the database lookup with nothing standing in the way.

### B.2 NEW, cryptographically-SIGNED VERP (authenticated cleartext, NOT encrypted)

Produced by `generate_verp_email()` [app/email_utils.py:L1438-L1464], quoted complete:

```python
def generate_verp_email(
    verp_type: VerpType, object_id: int, sender_domain: Optional[str] = None
) -> str:
    """Generates an email address with the verp type, object_id and domain encoded in the address
    and signed with hmac to prevent tampering
    """
    # Encoded as a list to minimize size of email address
    # Time is in minutes granularity and start counting on 2022-01-01 to reduce bytes to represent time
    data = [
        verp_type.value,
        object_id or 0,
        int((time.time() - VERP_TIME_START) / 60),
    ]
    json_payload = json.dumps(data).encode("utf-8")
    # Signing without itsdangereous because it uses base64 that includes +/= symbols and lower and upper case letters.
    # We need to encode in base32
    payload_hmac = hmac.new(
        config.VERP_EMAIL_SECRET.encode("utf-8"), json_payload, VERP_HMAC_ALGO
    ).digest()[:8]
    encoded_payload = base64.b32encode(json_payload).rstrip(b"=").decode("utf-8")
    encoded_signature = base64.b32encode(payload_hmac).rstrip(b"=").decode("utf-8")
    return "{}.{}.{}@{}".format(
        config.VERP_PREFIX,
        encoded_payload,
        encoded_signature,
        sender_domain or config.EMAIL_DOMAIN,
    ).lower()
```

**Precise terminology (correcting a common mischaracterization):** the identifier is **not encrypted**. The list `[verp_type, object_id, minutes]` is serialized to JSON and **base32-encoded** — a reversible, non-secret transformation — then **authenticated** with a truncated HMAC-`sha3-224` (8-byte digest) keyed by `VERP_EMAIL_SECRET`. Anyone can *read* the `object_id` out of a signed address by base32-decoding the payload field; what they cannot do is *forge* the HMAC. The correct description is therefore a **signed / authenticated cleartext payload**, not an encrypted one. Constants: `VERP_TIME_START = 1640995200` and `VERP_HMAC_ALGO = "sha3-224"` [app/email_utils.py:L68-L69]; `VERP_PREFIX = "sl"` and `VERP_MESSAGE_LIFETIME = 5 * 86400` (five days) [app/config.py:L499-L500]; `VERP_EMAIL_SECRET` [app/config.py:L502] (a runtime `RuntimeError` is raised if it is shorter than 32 characters, [app/config.py:L505-L507]).

**Observed round-trip (non-canonical, direct calls)** for the user's example id `369`; `get_verp_info_from_email()` returns a **2-tuple `(VerpType, object_id)`** — the timestamp is validated but not returned (full transcript: Phase K.2, `obs_A.out`):

```
generate_verp_email(bounce_forward, 369) = 'sl.lmycyibtgy4syibsgm4deobwgboq.b64xnu7ebjnwm@sl.local'
   get_verp_info_from_email() -> (<VerpType.bounce_forward: 0>, 369)
generate_verp_email(bounce_reply, 369) = 'sl.lmysyibtgy4syibsgm4deobwgboq.rnnobmm6d2vu6@sl.local'
   get_verp_info_from_email() -> (<VerpType.bounce_reply: 1>, 369)
generate_verp_email(transactional, 369) = 'sl.lmzcyibtgy4syibsgm4deobwgboq.mn6hurg5rweoc@sl.local'
   get_verp_info_from_email() -> (<VerpType.transactional: 2>, 369)
```

The shape is `sl.{base32-payload}.{base32-signature}@sl.local`; the signature field varies with the time bucket because it encodes minutes-since-`VERP_TIME_START`. The three VERP kinds come from the `VerpType` enum [app/models.py:L247-L250]:

```python
class VerpType(EnumE):
    bounce_forward = 0
    bounce_reply = 1
    transactional = 2
```

### B.3 Why both coexist

- The **OLD** format is classic plaintext **VERP** (Variable Envelope Return Path): the per-message identifier is embedded in the envelope in cleartext and recovered by a bare `int()`. It is trivially forgeable — anyone can write `bounce+{n}+@sl.local` for any integer `{n}`.
- The **NEW** format is the **signed / authenticated** VERP variant — functionally a **BATV**-style (Bounce Address Tag Validation) scheme: the (still-cleartext) identifier travels with an unforgeable HMAC tag plus a timestamp, so a forged or far-future address fails verification. See Phase H for the framing against established practice, and Phase F for the observed one-sided (future-only) expiry behavior.

The security-relevant consequence of this split — that the two formats are routed to the *same* branch but validated *differently* — is the subject of Phase C.

---

## Phase C — Objective 2: Routing (all four branches, all observed)

### C.1 The canonical entry chain and the recipient dispatch

An inbound message enters at `MailHandler.handle_DATA()` [email_handler.py:L2289], which calls `_handle()` [email_handler.py:L2335], which calls the module-level dispatcher `handle()` [email_handler.py:L1945]. Inside `handle()`, the VERP routing block [email_handler.py:L2034-L2116] begins by computing, at [email_handler.py:L2035]:

```python
verp_info = get_verp_info_from_email(rcpt_tos[0])
```

`verp_info` is a `(VerpType, object_id)` tuple for a **valid signed** recipient, or `None` for anything else (unsigned addresses, or signed addresses that fail verification). The four branches are then tried in order.

### C.2 Transactional branch [email_handler.py:L2038-L2054]

Condition: the recipient matches the transactional prefix/suffix (`TRANSACTIONAL_BOUNCE_PREFIX = "transactional+"` [app/config.py:L113-L114]) **or** `verp_info[0] == VerpType.transactional`. This branch has **no `EmailLog.get()` and no `E512`** — the id is never looked up. It returns `status.E205` [email_handler.py:L2047] for a bounce, `status.E206` [email_handler.py:L2052] for an automatic out-of-office, else raises `VERPTransactional` [email_handler.py:L2054].

**Observed, canonical** (signed transactional address `sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local`; full transcript Phase K.6, `obs_E.out`):

```
OOO (Auto-Submitted) -> handle()=RETURNED '250 SL E206 Out of office' | handle_DATA()='250 SL E206 Out of office'
BOUNCE (<>,mp/report)-> handle()=RETURNED '250 SL E205 bounce handled' | handle_DATA()='250 SL E205 bounce handled'
NEITHER (non-bounce) -> handle()=RAISED VERPTransactional(VERPTransactional ) | handle_DATA()='250 SL E213 Unknown email ignored'
```

=> The transactional branch **does not enumerate**: because there is no id lookup, an invalid id yields the same `E205`/`E206`/`E213` as any other, so it cannot distinguish existent from non-existent identifiers. (This is also the **only** branch that returns `E206` for an out-of-office message — see C.3.)

### C.3 Forward-bounce branch [email_handler.py:L2057-L2074] — the crux

Condition (note the `or`): **either** the recipient is an OLD unsigned `bounce+{id}+@{domain}` address **or** it is a valid signed forward address. Quoted complete:

```python
    if (
        len(rcpt_tos) == 1
        and rcpt_tos[0].startswith(BOUNCE_PREFIX)
        and rcpt_tos[0].endswith(BOUNCE_SUFFIX)
    ) or (verp_info and verp_info[0] == VerpType.bounce_forward):
        email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(rcpt_tos[0])
        email_log = EmailLog.get(email_log_id)

        if not email_log:
            LOG.w("No such email log")
            return status.E512

        if is_bounce(envelope, msg):
            return handle_bounce(envelope, email_log, msg)
        elif is_automatic_out_of_office(msg):
            handle_out_of_office_forward_phase(email_log, envelope, msg, rcpt_tos)
        else:
            raise VERPForward
```

Two facts make this branch the heart of the oracle:

1. **Both** address formats hit this **same** branch (the OLD via the `startswith/endswith` test, the NEW via `verp_info[0] == bounce_forward`).
2. **The existence check precedes the bounce gate.** `EmailLog.get()` and the `return status.E512` fire *before* the `is_bounce()` test. So a probe reveals whether the id exists regardless of whether it looks like a real bounce.

Note also that the out-of-office sub-branch here calls `handle_out_of_office_forward_phase()` **without a `return`**, so it falls through into the ordinary forward-delivery logic below the branch — this branch never returns `E206` (that code comes only from the transactional branch, C.2). The **observed** canonical fall-through outcome is `250 SL E214 Unauthorized for using reverse alias` for a valid id (the reverse-alias authorization check downstream rejects the unauthorized sender); an invalid id still returns `550 SL E512 No such email log` because the existence check fires first. See the out-of-office block below and the full canonical transcript in Phase K.10 (`gb_run1.out`/`gb_run2.out`).

**Why the two formats get different validation despite sharing the branch:** the id `email_log_id` comes from `verp_info[1]` for the signed form — a value produced only **after** `get_verp_info_from_email()` has already verified the HMAC and timestamp — whereas for the unsigned form it comes from `parse_id_from_bounce(rcpt_tos[0])`, a bare `int()` with **no** verification. The unsigned id is attacker-chosen; the signed id is cryptographically vouched-for.

**Observed, canonical** (live fixture: `user.id=479`, valid forward ids `385/386`, invalid `99999999999999`; full transcript Phase K.3, `obs_B.out`):

```
FWD OLD-unsigned INVALID non-bounce : rcpt='bounce+99999999999999+@sl.local'
   handle()=RETURNED '550 SL E512 No such email log' | handle_DATA()='550 SL E512 No such email log'
FWD OLD-unsigned VALID   non-bounce : rcpt='bounce+385+@sl.local'
   handle()=RAISED VERPForward | handle_DATA()='250 SL E213 Unknown email ignored'
FWD OLD-unsigned VALID   bounce     : mail_from='<>' rcpt='bounce+386+@sl.local'
   handle()=RETURNED '250 SL E211 Bounce Forward phase handled' | handle_DATA()='250 SL E211 Bounce Forward phase handled'
FWD SIGNED       INVALID non-bounce : rcpt='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.d2u7yvh73aezm@sl.local'
   handle()=RETURNED '550 SL E512 No such email log' | handle_DATA()='550 SL E512 No such email log'
FWD SIGNED       VALID   non-bounce : rcpt='sl.lmycyibtha2syibsgm4deobwgboq.mkvkzf33xtabo@sl.local'
   handle()=RAISED VERPForward | handle_DATA()='250 SL E213 Unknown email ignored'
FWD SIGNED       VALID   bounce     : mail_from='<>' rcpt='sl.lmycyibtha3syibsgm4deobwgboq.luaymr7wkenmi@sl.local'
   handle()=RETURNED '250 SL E211 Bounce Forward phase handled' | handle_DATA()='250 SL E211 Bounce Forward phase handled'
```

**Observed, canonical — forward OUT-OF-OFFICE fall-through** (`Auto-Submitted: auto-replied`, non-`<>` sender, `text/plain`; `is_bounce`=`False`, `is_automatic_out_of_office`=`True`; every value produced by `handle_DATA()`; full transcript Phase K.10, `gb_run1.out`/`gb_run2.out`):

```
FWD OLD    VALID   OOO : rcpt='bounce+349+@sl.local'                                 handle_DATA()='250 SL E214 Unauthorized for using reverse alias'
FWD OLD    INVALID OOO : rcpt='bounce+99999999999999+@sl.local'                      handle_DATA()='550 SL E512 No such email log'
FWD SIGNED VALID   OOO : rcpt='sl.lmycyibtgq4syibsgm4dgmzugzoq.gasefkay7hte6@sl.local' handle_DATA()='250 SL E214 Unauthorized for using reverse alias'
FWD SIGNED INVALID OOO : rcpt='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local' handle_DATA()='550 SL E512 No such email log'
```

=> The forward OOO sub-branch has the **same `E512`-on-invalid divergence** as the non-bounce probe: the valid id falls through to `250 SL E214 Unauthorized for using reverse alias` while the invalid id is rejected by the earlier existence check with `550 SL E512 No such email log`. So the enumeration oracle is observable through an out-of-office message as well.

### C.4 Reply-bounce branch [email_handler.py:L2077-L2098]

Parallel to the forward branch. Condition: recipient starts with `f"{BOUNCE_PREFIX_FOR_REPLY_PHASE}+"` (`BOUNCE_PREFIX_FOR_REPLY_PHASE = "bounce_reply"` [app/config.py:L108-L110]) **or** `verp_info[0] == VerpType.bounce_reply`. Same `EmailLog.get()` -> `E512`-if-missing [email_handler.py:L2087] -> `handle_bounce` if `is_bounce` [email_handler.py:L2091] -> else `raise VERPReply` [email_handler.py:L2095].

**Observed, canonical** (live fixture reply ids `388/389`; full transcript Phase K.3, `obs_B.out`):

```
REP OLD-unsigned INVALID bounce     : mail_from='<>' rcpt='bounce_reply+99999999999999+@sl.local'
   handle()=RETURNED '550 SL E512 No such email log' | handle_DATA()='550 SL E512 No such email log'
REP SIGNED       INVALID non-bounce : rcpt='sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.cidobv3wlys4e@sl.local'
   handle()=RETURNED '550 SL E512 No such email log' | handle_DATA()='550 SL E512 No such email log'
REP SIGNED       VALID   bounce     : mail_from='<>' rcpt='sl.lmysyibtha4cyibsgm4deobwgboq.6t6t7b3x2o43g@sl.local'
   handle()=RETURNED '250 SL E212 Bounce Reply phase handled' | handle_DATA()='250 SL E212 Bounce Reply phase handled'
REP SIGNED       VALID   non-bounce : rcpt='sl.lmysyibtha4syibsgm4deobwgboq.4exnsuakl4jf2@sl.local'
   handle()=RAISED VERPReply | handle_DATA()='250 SL E213 Unknown email ignored'
```

As on the forward branch, the reply out-of-office sub-branch calls `handle_out_of_office_reply_phase()` **without a `return`** [email_handler.py:L2092-L2093], so it falls through into the ordinary reply-delivery logic. The **observed** canonical fall-through outcome differs from the forward branch: a valid id returns `250 Message accepted for delivery` (the reply path accepts the message), while an invalid id still returns `550 SL E512 No such email log`.

**Observed, canonical — reply OUT-OF-OFFICE fall-through** (`Auto-Submitted: auto-replied`, non-`<>` sender, `text/plain`; every value produced by `handle_DATA()`; full transcript Phase K.10, `gb_run1.out`/`gb_run2.out`):

```
REP OLD    VALID   OOO : rcpt='bounce_reply+350+@sl.local'                           handle_DATA()='250 Message accepted for delivery'
REP OLD    INVALID OOO : rcpt='bounce_reply+99999999999999+@sl.local'                handle_DATA()='550 SL E512 No such email log'
REP SIGNED VALID   OOO : rcpt='sl.lmysyibtguycyibsgm4dgmzugzoq.awmia3ereye3u@sl.local' handle_DATA()='250 Message accepted for delivery'
REP SIGNED INVALID OOO : rcpt='sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local' handle_DATA()='550 SL E512 No such email log'
```

=> The reply branch has the **same `E512`/`E213` divergence** as the forward branch on the non-bounce probe; the only bounce-processing difference is `E212` (reply) vs `E211` (forward). Its out-of-office fall-through returns `250 Message accepted for delivery` (valid) vs `550 SL E512 No such email log` (invalid), so the reply OOO path leaks id existence too.

### C.5 iCloud branch [email_handler.py:L2100-L2116] — most readily exploitable

This branch keys off the **envelope `mail_from`** (not the recipient), computing `verp_info = get_verp_info_from_email(mail_from[0])` at [email_handler.py:L2101], and — critically — it calls `handle_bounce` **unconditionally, with no `is_bounce` gate** [email_handler.py:L2116].

**Observed (non-canonical, direct `handle()` call)** (bounce address in the envelope `mail_from`, a real alias `weeper_became557@sl.local` in the recipient, ordinary `text/plain` **non-bounce** message; full transcript Phase K.6, `obs_E.out`). These rows call the module-level `handle()` directly and are labelled non-canonical because they bypass the `handle_DATA` -> `_handle` entry point; the canonical `handle_DATA` confirmation follows immediately below:

```
iCloud VALID id  : mail_from='bounce+397+@sl.local' rcpt='weeper_became557@sl.local' -> handle()=RETURNED '250 SL E211 Bounce Forward phase handled'
iCloud INVALID id: mail_from='bounce+99999999999999+@sl.local' rcpt='weeper_became557@sl.local' -> handle()=RETURNED '550 SL E512 No such email log'
```

**Observed, canonical (via `handle_DATA`)** — the same iCloud probe driven through the real entry point `MailHandler.handle_DATA()` on a fresh fixture (full transcript Phase K.10, `gb_run1.out`/`gb_run2.out`):

```
iCloud OLD VALID   mail_from : mail_from='bounce+353+@sl.local'            rcpt='ramify_hoopla407@sl.local' -> handle_DATA()='250 SL E211 Bounce Forward phase handled'
iCloud OLD INVALID mail_from : mail_from='bounce+99999999999999+@sl.local' rcpt='ramify_hoopla407@sl.local' -> handle_DATA()='550 SL E512 No such email log'
```

=> The canonical `handle_DATA` result reproduces the direct-`handle()` outcome exactly: a valid id in the envelope `mail_from` yields `250 SL E211 Bounce Forward phase handled`, an invalid id yields `550 SL E512 No such email log` — the `E211`/`E512` divergence is confirmed at the real entry point.

And with a **signed** address in `mail_from` (full transcript Phase K.8, `obs_H.out`):

```
signed mail_from = 'sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local'  (mail_from[0]='s', the FIRST CHARACTER)
iCloud SIGNED mail_from -> handle_DATA() (run1) = '250 Message accepted for delivery'
iCloud SIGNED mail_from -> handle_DATA() (run2) = '250 Message accepted for delivery'
```

The signed row exposes a **latent bug**: [email_handler.py:L2101] calls `get_verp_info_from_email(mail_from[0])`, and `mail_from[0]` is the **first character** of the string (observed here as `'s'`), not the first element of a list. It therefore never yields a valid tuple, so a *signed* address can never satisfy the `verp_info[0] == bounce_forward` half of the condition and falls through to `250 Message accepted for delivery` — only the OLD unsigned form can reach this branch (observed: the signed `mail_from` returns `E200` while the unsigned `mail_from` returns `E211`/`E512`).

=> The iCloud branch is the **most readily exploitable** oracle surface: it leaks via the attacker-controlled envelope sender **without** requiring `is_bounce` to be satisfied (there is no gate — `handle_bounce` runs unconditionally at [email_handler.py:L2116]), and only the OLD unsigned format can reach it.

---

## Phase D — Objectives 3, 5 & 7: The oracle (feasibility, exact responses, richness, ordering)

### D.1 The core matrix (Objectives 3 & 5)

Unsigned `bounce+{id}+@sl.local`, ordinary **non-bounce** message, forward-bounce branch. Every string below was captured verbatim and was **stable** across at least two runs (live fixture: valid id `385`, invalid id `99999999999999`; full transcript Phase K.3, `obs_B.out`):

| id | `handle()` | `_handle()` | `handle_DATA()` |
|----|-----------|-------------|-----------------|
| **INVALID** (`99999999999999`) | RETURN `550 SL E512 No such email log` | RETURN `550 SL E512 No such email log` | RETURN `550 SL E512 No such email log` |
| **VALID** (`385`) | RAISE `VERPForward` | RAISE `VERPForward` | RETURN `250 SL E213 Unknown email ignored` |

**The oracle, stated plainly:** an invalid id returns **`550 SL E512 No such email log`**; a valid id probed with a non-bounce message returns **`250 SL E213 Unknown email ignored`**. These differ at both the numeric SMTP level (`550` vs `250`) and the SimpleLogin sub-code (`E512` vs `E213`).

**External-attacker feasibility (Objective 3):** the only inputs required are the SMTP envelope recipient (`RCPT TO`) — set to `bounce+{n}+@sl.local` — and an arbitrary message body. Both are fully under the control of any external sender who can deliver a message to the SimpleLogin MX. No account, no authentication, and no valid signature is needed. Reading the returned status string therefore tells the attacker whether the integer `{n}` is a live `email_log_id`.

The signed forward format shares the branch and behaves identically **for a valid id** (live valid id `385` signed; full transcript Phase K.3):

| Signed forward, non-bounce | `handle()` | `handle_DATA()` |
|----|-----------|-----------------|
| INVALID id | RETURN `550 SL E512 No such email log` | RETURN `550 SL E512 No such email log` |
| VALID id | RAISE `VERPForward` | RETURN `250 SL E213 Unknown email ignored` |

So the `E512`/`E213` divergence is a property of the forward/reply/iCloud **id-lookup**, not of the address format itself. What differs is *reachability*: a forged **signed** address can never get here (Phase F), whereas a forged **unsigned** address always can.

### D.2 The ordering that makes even a non-bounce probe leak (Objective 7, part 1)

Because `EmailLog.get()` / `return status.E512` [email_handler.py:L2063-L2067] execute **before** the `is_bounce()` gate [email_handler.py:L2069], the existence of the id is decided *before* the code cares whether the message is a genuine bounce. Consequently a plain, non-bounce probe already distinguishes existent from non-existent ids — the attacker does **not** need to craft a real DSN. This ordering is confirmed on the bounce path too (from `obs_B.out`):

- OLD **valid** id `386` + `is_bounce` (active user) -> `250 SL E211 Bounce Forward phase handled`
- OLD **invalid** id `99999999999999` + `is_bounce` -> `550 SL E512 No such email log` (the existence check still fires first)

### D.3 Richness — the codes reveal more than existence (Objective 7, part 2)

A valid id whose owning **user is inactive** produces a third, distinct response, emitted by `handle_bounce()` [email_handler.py:L1869-L1871], quoted complete:

```python
    if not email_log.user.is_active():
        LOG.d(f"User {email_log.user} is not active")
        return status.E510
```

`E510` is the literal string `"550 SL E510 so such user"` [app/email/status.py:L47] — the source's typo ("so such user") is preserved here exactly as shipped. The mechanism is `User.is_active()` [app/models.py:L766-L769], quoted complete:

```python
    def is_active(self) -> bool:
        if self.delete_on is None:
            return True
        return self.delete_on < arrow.now()
```

**Observed, canonical** (fresh fixture whose user is set inactive; signed valid forward id, `is_bounce` true; from `obs_B.out`, Phase K.3):

```
FWD SIGNED VALID bounce, INACTIVE user -> handle()=RETURNED '550 SL E510 so such user' | handle_DATA()='550 SL E510 so such user'
```

=> **Three-way discrimination:** *nonexistent id* -> `550 SL E512 No such email log`; *existent but inactive user* -> `550 SL E510 so such user`; *existent, active* -> `250 SL E211 Bounce Forward phase handled` (bounce) or `250 SL E213 Unknown email ignored` (non-bounce). The oracle leaks not just whether an `email_log_id` exists, but also a fact about the state of the account that owns it.

### D.4 State transitions on the bounce path (Objective 7 / mutation evidence)

When a valid id on the forward branch is probed with a **genuine bounce** and the user is active, `handle_bounce_forward_phase()` [email_handler.py:L1432-L1520] mutates persistent state: it creates a `Bounce` row and a `RefusedEmail` row and sets `email_log.bounced = True`, `email_log.refused_email_id`, and `email_log.bounced_mailbox_id`, then commits. Per Phase A.5, these are **committed mutations to the ephemeral test DB** (not rolled back in SQLAlchemy 1.3.24).

**Observed, canonical (via `handle_DATA`)** — captured **before** and **after** driving the genuine forward-phase bounce (`mail_from='<>'`, signed `bounce_forward`, `multipart/report`) through the real entry point `MailHandler.handle_DATA()` on a fresh fixture, run **twice**. The concrete sequence-backed ids advance run-to-run (a property of the accumulated ephemeral test DB), but the state semantics and deltas are identical. Full transcript: Phase K.10 (`gb_run1.out`/`gb_run2.out`):

```
--- run 1 ---
BEFORE : email_log(id=354).bounced=False refused_email_id=None bounced_mailbox_id=None | Bounce_rows=113 RefusedEmail_rows=114
HANDLER: handle_DATA() returned '250 SL E211 Bounce Forward phase handled'
AFTER  : email_log(id=354).bounced=True refused_email_id=118 bounced_mailbox_id=534 | Bounce_rows=114 RefusedEmail_rows=115
DELTA  : Bounce_rows +1, RefusedEmail_rows +1
--- run 2 ---
BEFORE : email_log(id=360).bounced=False refused_email_id=None bounced_mailbox_id=None | Bounce_rows=115 RefusedEmail_rows=116
HANDLER: handle_DATA() returned '250 SL E211 Bounce Forward phase handled'
AFTER  : email_log(id=360).bounced=True refused_email_id=120 bounced_mailbox_id=537 | Bounce_rows=116 RefusedEmail_rows=117
DELTA  : Bounce_rows +1, RefusedEmail_rows +1
```

For completeness, the same transition was **also** observed earlier via a **(non-canonical, direct `handle()` call)** on a fresh fixture (`email_log id=400`), with matching semantics and deltas; full transcript Phase K.7 (`obs_F.out`):

```
BEFORE : email_log(id=400).bounced=False refused_email_id=None bounced_mailbox_id=None | Bounce_rows=29 RefusedEmail_rows=30
HANDLER: handle() returned '250 SL E211 Bounce Forward phase handled'
AFTER  : email_log(id=400).bounced=True refused_email_id=47 bounced_mailbox_id=578 | Bounce_rows=30 RefusedEmail_rows=31
DELTA  : Bounce_rows +1, RefusedEmail_rows +1 (committed to ephemeral test DB; repo untouched)
```

So a *successful* bounce probe of a valid id is observable **twice**: once via the `250 SL E211 Bounce Forward phase handled` response, and once via the `bounced`/`refused_email_id`/`bounced_mailbox_id` column transitions and the `+1`/`+1` row growth. (The enumeration leak itself needs neither — a non-bounce probe suffices, D.1/D.2 — but the state change confirms the id resolved to a real, mutable `EmailLog`.)

### D.5 Summary of the leak

- **Feasibility (Obj 3):** trivial for any external sender; only the envelope recipient and a body are needed.
- **Exact responses (Obj 5):** `550 SL E512 No such email log` (invalid) vs `250 SL E213 Unknown email ignored` (valid, non-bounce) vs `250 SL E211 Bounce Forward phase handled` (valid, bounce, active) vs `550 SL E510 so such user` (valid, inactive user).
- **Richness + ordering (Obj 7):** more than existence (inactive-user `550 SL E510 so such user`), and the existence check runs before the bounce gate so a non-bounce probe suffices.

---

## Phase E — Objective 6: `is_bounce()` and its spoofability

### E.1 The criteria

`is_bounce()` [email_handler.py:L1813-L1818] classifies a message as a Delivery Status Notification using exactly two inputs, quoted complete:

```python
def is_bounce(envelope: Envelope, msg: Message):
    """Detect whether an email is a Delivery Status Notification"""
    return (
        envelope.mail_from == "<>"
        and msg.get_content_type().lower() == "multipart/report"
    )
```

### E.2 Observed truth table (non-canonical, direct calls)

These four rows come from calling `is_bounce()` directly (labeled non-canonical because they bypass the handler chain); full transcript Phase K.6 (`obs_E.out`):

```
is_bounce(mail_from='<>'                  , Content-Type='multipart/report') = True
is_bounce(mail_from='<>'                  , Content-Type='text/plain'      ) = False
is_bounce(mail_from='attacker@evil.example', Content-Type='multipart/report') = False
is_bounce(mail_from='attacker@evil.example', Content-Type='text/plain'      ) = False
```

Only the exact combination `("<>", multipart/report)` returns `True`.

### E.3 Both inputs are attacker-controllable

- `envelope.mail_from == "<>"` is the SMTP **`MAIL FROM`** command value. A null return-path (`MAIL FROM:<>`) is a normal, sender-chosen envelope value.
- `msg.get_content_type()` is read from the **`Content-Type`** header of the message the sender supplies.

Neither is derived from any server-side trust decision, so an external sender can trivially set both and satisfy `is_bounce()`, reaching `handle_bounce` on the forward and reply branches. (On the iCloud branch, Phase C.5, `is_bounce` is not even required — `handle_bounce` runs unconditionally.)

### E.4 The pass/fail effect on the response (canonical)

- When `is_bounce` **passes** on a **valid** id, the response advances from the enumeration-only path (`raise VERPForward` -> `E213`) to the bounce-processing codes: `250 SL E211 Bounce Forward phase handled` (forward), `250 SL E212 Bounce Reply phase handled` (reply), or `550 SL E510 so such user` if the user is inactive. All of these were observed canonically in Phase C.3/C.4 and Phase D.3.
- When `is_bounce` **fails** (non-bounce probe) on a valid id, `handle()` raises `VERPForward`/`VERPReply` -> `handle_DATA()` `250 SL E213 Unknown email ignored`.

Crucially, **either way the existence check has already fired** (Phase D.2): an invalid id returns `550 SL E512 No such email log` before `is_bounce` is consulted. So satisfying — or not satisfying — `is_bounce` changes only the *valid-id* response; the existent-vs-nonexistent leak is observable regardless of whether the probe is dressed up as a bounce.

---

## Phase F — Objective 4: The security boundary (what slips through without crypto validation)

### F.1 The two paths, precisely

- **UNSIGNED** `bounce+{id}+@sl.local`: the id is produced by `parse_id_from_bounce()` — a bare `int()` [app/email_utils.py:L1258-L1259] — and handed straight to `EmailLog.get(id)` [email_handler.py:L2063]. There is **no signature check and no timestamp check** anywhere on this path.
- **SIGNED**: `get_verp_info_from_email()` [app/email_utils.py:L1467-L1498] returns `None` on any of **six** conditions, quoted complete:

```python
def get_verp_info_from_email(email: str) -> Optional[Tuple[VerpType, int]]:
    """This method processes the email address, checks if it's a signed verp email generated by us to receive bounces
    and extracts the type of verp email and associated email log id/transactional email id stored as object_id
    """
    idx = email.find("@")
    if idx == -1:
        return None
    username = email[:idx]
    fields = username.split(".")
    if len(fields) != 3 or fields[0] != config.VERP_PREFIX:
        return None
    try:
        padding = (8 - (len(fields[1]) % 8)) % 8
        payload = base64.b32decode(fields[1].encode("utf-8").upper() + (b"=" * padding))
        padding = (8 - (len(fields[2]) % 8)) % 8
        signature = base64.b32decode(
            fields[2].encode("utf-8").upper() + (b"=" * padding)
        )
    except binascii.Error:
        return None
    expected_signature = hmac.new(
        config.VERP_EMAIL_SECRET.encode("utf-8"), payload, VERP_HMAC_ALGO
    ).digest()[:8]
    if expected_signature != signature:
        return None
    data = json.loads(payload)
    # verp type, object_id, time
    if len(data) != 3:
        return None
    if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60:
        return None
    return VerpType(data[0]), data[1]
```

The **six** `None`-returning conditions, in source order, are:

1. **No `@` in the address** — `if idx == -1: return None` [app/email_utils.py:L1472-L1473].
2. **Wrong field count or wrong prefix** — `if len(fields) != 3 or fields[0] != config.VERP_PREFIX: return None` [app/email_utils.py:L1476-L1477].
3. **base32 decode failure** on the payload or signature — `except binascii.Error: return None` [app/email_utils.py:L1485-L1486].
4. **HMAC mismatch** — `if expected_signature != signature: return None` [app/email_utils.py:L1490-L1491].
5. **Decoded JSON list not length 3** — `if len(data) != 3: return None` [app/email_utils.py:L1494-L1495].
6. **Timestamp beyond the lifetime window** (future/expired) — `if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60: return None` [app/email_utils.py:L1496-L1497].

**A distinct exception path (E404, not `None`/E515):** the `json.loads(payload)` call at [app/email_utils.py:L1492] sits **outside** the `try/except binascii.Error` block (which only guards the base32 decode at L1478-L1486). So if a *verified* payload is not valid JSON, `json.loads` **raises** rather than returning `None`; that exception propagates out of `get_verp_info_from_email()` and is caught by the generic handler in `handle_DATA()` [email_handler.py:L2331-L2332], yielding `421 SL E404 Unexpected error - Retry later`. This is different from condition 5 above (`len(data) != 3`), which returns `None` and — for a signed-looking recipient — surfaces downstream as `550 SL E515 Email not exist`. In practice conditions 5 and the `json.loads` raise are only reachable for a payload that already passed the HMAC check at condition 4 (i.e. one our own signer produced), because a forged address cannot satisfy the HMAC; an external attacker's malformed signed-looking address is rejected at conditions 1-4 (observed as `E515`; see Phase K.4 and the verifier-shape observations in Phase K.11).

### F.2 The expiry test is ONE-SIDED (reported exactly, contradiction resolved)

The single comparison at [app/email_utils.py:L1496] is:

```python
    if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60:
        return None
```

This rejects a timestamp only when it is too far in the **future** (more than `VERP_MESSAGE_LIFETIME` = 5 days ahead of "now"). There is **no lower bound** — an *old* timestamp is never rejected. So a captured/old signed address remains valid indefinitely into the past. This is confirmed canonically (see F.3): a `-6d`-dated signed address is **still accepted** and produces the same outcome as a freshly-signed one, whereas a `+6d`-dated one is rejected. This corrects any impression that "stale" signed addresses are refused: only **future**-dated ones are.

### F.3 Driving GOOD / CORRUPTED / FUTURE / OLD signed addresses through the canonical handler

This is the decisive boundary observation. Signed addresses were produced by the software's own `generate_verp_email()` (the FUTURE/OLD variants by patching the signer's clock so the software itself timestamps them `+6d`/`-6d`), then each was driven through the canonical `handle()`/`handle_DATA()`. Live fixture: `valid_forward_email_log_id=392`, `VERP_MESSAGE_LIFETIME=5.0 days`. Stable across two runs; full transcript Phase K.4 (`obs_C.out`):

```
SIGNED GOOD (just signed)          address='sl.lmycyibthezcyibsgm4deobwgfoq.a5s77tdbqy7i6@sl.local'
   get_verp_info_from_email() [NON-CANONICAL helper] = (<VerpType.bounce_forward: 0>, 392)
   handle()      [CANONICAL, non-bounce probe] : RAISED VERPForward
   handle_DATA() [CANONICAL] : '250 SL E213 Unknown email ignored'

SIGNED CORRUPTED signature         address='sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local'
   get_verp_info_from_email() [NON-CANONICAL helper] = None
   handle()      [CANONICAL, non-bounce probe] : RETURNED '550 SL E515 Email not exist'
   handle_DATA() [CANONICAL] : '550 SL E515 Email not exist'

SIGNED FUTURE-dated +6d            address='sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local'
   get_verp_info_from_email() [NON-CANONICAL helper] = None
   handle()      [CANONICAL, non-bounce probe] : RETURNED '550 SL E515 Email not exist'
   handle_DATA() [CANONICAL] : '550 SL E515 Email not exist'

SIGNED OLD-dated -6d (in the past) address='sl.lmycyibthezcyibsgm3timrsgfoq.bdfrd5kda2hog@sl.local'
   get_verp_info_from_email() [NON-CANONICAL helper] = (<VerpType.bounce_forward: 0>, 392)
   handle()      [CANONICAL, non-bounce probe] : RAISED VERPForward
   handle_DATA() [CANONICAL] : '250 SL E213 Unknown email ignored'
```

`E515` is `"550 SL E515 Email not exist"` [app/email/status.py:L51]. Reading these four rows:

- **GOOD** (valid HMAC, current time): verification succeeds, `verp_info=(bounce_forward, 392)`, so the message **enters the forward branch**; because id `392` exists and the probe is non-bounce, `handle()` raises `VERPForward` -> `250 SL E213 Unknown email ignored`. This proves a good signature **does** reach the email-log lookup.
- **CORRUPTED** (first base32 character of the signature field flipped from `a` to `b`: `sl.lmycyibthezcyibsgm4deobwgfoq.a5s77tdbqy7i6@sl.local` -> `sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local`): HMAC mismatch -> `verp_info=None` -> the message **never enters the bounce branch**; it falls through to normal alias handling and returns `550 SL E515 Email not exist`.
- **FUTURE +6d**: expiry test fails -> `verp_info=None` -> same `550 SL E515 Email not exist` diversion.
- **OLD -6d**: expiry test passes (no lower bound) -> `verp_info=(bounce_forward, 392)` -> honored exactly like GOOD -> `250 SL E213 Unknown email ignored`.

### F.4 The boundary

- A **GOOD** signature is required to reach the email-log lookup on the signed path — and when present, it *does* reach it (observed: GOOD -> forward branch -> `VERPForward`/`E213` for the existing id `392`). So the signed path reaches the oracle **only** after cryptographic verification succeeds.
- A **corrupted or future-expired** signature makes `verp_info == None`, which **diverts the probe away from the oracle** entirely (to normal alias lookup -> `550 SL E515 Email not exist`), so it never reaches the `E512`/`E213`/`E510` email-log responses.
- An **old** signature (past-dated) is **still accepted** (one-sided expiry, F.2), so it reaches the oracle just like a fresh one.
- The **UNSIGNED** OLD path has **no such cryptographic gate**: `parse_id_from_bounce` -> `EmailLog.get(id)` runs unconditionally.

**This is the security boundary (Objective 4):** the thing that slips through without cryptographic validation is the attacker-chosen `email_log_id` on the unsigned `bounce+{id}+@domain` path — it reaches `EmailLog.get(id)` directly, whereas the signed path's HMAC gate turns a forged probe into a benign `550 SL E515 Email not exist` that never touches the email-log lookup. The signed path's *timestamp* gate, however, is one-sided and constrains only future-dated addresses.

---

## Phase G — Edge condition: SPF-based `E216` suppression

### G.1 The rewrite

`_handle()` [email_handler.py:L2353-L2365] overwrites a returned `5XX` status with `E216` when the return-path failed SPF, quoted complete:

```python
            return_status = handle(envelope, msg)
            elapsed = time.time() - start
            # Only bounce messages if the return-path passes the spf check. Otherwise black-hole it.
            spamd_result = SpamdResult.extract_from_headers(msg)
            if return_status[0] == "5":
                if spamd_result and spamd_result.spf in (
                    SPFCheckResult.fail,
                    SPFCheckResult.soft_fail,
                ):
                    LOG.i(
                        "Replacing 5XX to 216 status because the return-path failed the spf check"
                    )
                    return_status = status.E216
```

The SPF verdict is parsed from the rspamd `X-Spamd-Result` header by `SpamdResult.extract_from_headers()` [app/handler/spamd_result.py:L76]. In `SPFCheckResult` [app/handler/spamd_result.py:L32-L46], `fail` and `soft_fail` share the same enum value (`1`), so `soft_fail` is an **alias** of `fail`; both `R_SPF_FAIL` and `R_SPF_SOFTFAIL` therefore trigger the rewrite, while `R_SPF_ALLOW` does not:

```python
class SPFCheckResult(EnumE):
    allow = 0
    fail = 1
    soft_fail = 1
    neutral = 2
    temp_error = 3
    not_available = 4
    perm_error = 5
```

### G.2 Observed suppression matrix — one row per distinct verdict (not merged)

Each SPF verdict is reported as its **own** row (fail, soft-fail, allow, and absent are distinct conditions; they are not merged). Unsigned **invalid** id, non-bounce, driven through `_handle()`, with the SPF token injected on its own continuation line of `X-Spamd-Result` (the canonical `tests/example_emls/5xx_overwrite_spf.eml` template; absent-header uses `tests/example_emls/no_spamd_header.eml`). Stable across two runs; full transcript Phase K.5 (`obs_D.out`):

| # | Address | id | SPF verdict | `_handle()` result |
|---|---------|----|-----|--------------------|
| 1 | OLD unsigned `bounce+99999999999999+@sl.local` | invalid | `R_SPF_FAIL` | `250 SL E216 Handled spf policy` |
| 2 | OLD unsigned | invalid | `R_SPF_SOFTFAIL` | `250 SL E216 Handled spf policy` |
| 3 | OLD unsigned | invalid | `R_SPF_ALLOW` | `550 SL E512 No such email log` |
| 4 | OLD unsigned | invalid | *(no `X-Spamd-Result` header)* | `550 SL E512 No such email log` |
| 5 | NEW signed forward | invalid | `R_SPF_FAIL` | `250 SL E216 Handled spf policy` |
| 6 | NEW signed forward | invalid | `R_SPF_SOFTFAIL` | `250 SL E216 Handled spf policy` |
| 7 | NEW signed forward | invalid | `R_SPF_ALLOW` | `550 SL E512 No such email log` |
| 8 | NEW signed forward | invalid | *(no header)* | `550 SL E512 No such email log` |

Inactive-user **valid** id, `is_bounce` (which returns `E510`, a `5XX`, so it *is* subject to the rewrite; live id `393`, `u2.is_active()=False`):

| # | id | SPF verdict | `_handle()` result |
|---|----|-----|--------------------|
| 9 | valid (inactive user) `393` | `R_SPF_FAIL` | `250 SL E216 Handled spf policy` (the `E510` is silenced) |
| 10 | valid (inactive user) `393` | `R_SPF_ALLOW` | `550 SL E510 so such user` (preserved when SPF passes) |
| 11 | valid (inactive user) `393` | *(no header)* | `550 SL E510 so such user` (preserved) |

### G.3 The asymmetry that lets the oracle survive SPF-fail

The **valid** non-bounce id is **never** suppressed, because it **raises `VERPForward` before `_handle()`'s rewrite runs** — the rewrite only inspects a *returned* status (`return_status[0] == "5"`), not a *raised* exception. Driven through `handle_DATA()` (which maps the raised exception to `E213`), live valid id `395`; stable across two runs; full transcript Phase K.5 (`obs_D2.out`):

| SPF verdict | INVALID id, non-bounce -> `handle_DATA()` | VALID id, non-bounce -> `handle_DATA()` |
|-------------|-------------------------------------------|------------------------------------------|
| `R_SPF_FAIL` | `250 SL E216 Handled spf policy` | `250 SL E213 Unknown email ignored` |
| `R_SPF_ALLOW` | `550 SL E512 No such email log` | `250 SL E213 Unknown email ignored` |
| *(no header)* | `550 SL E512 No such email log` | `250 SL E213 Unknown email ignored` |

Under `R_SPF_FAIL` the invalid id yields `E216` and the valid id yields `E213` — **still two distinct strings**. The rewrite therefore narrows the numeric divergence (`550`->`250`) but does not eliminate the differential response.

### G.4 Two critical nuances

1. **The SPF verdict is not attacker-chosen.** It is read from the `X-Spamd-Result` header that the receiving rspamd/MTA injects — it is *not* something the external sender sets on the SMTP envelope. Whether the rewrite fires therefore depends on the receiver's SPF verdict for the probe's return-path, not on a flag the attacker can toggle at will.
2. **The rewrite applies only to a returned `5XX`, not to a raised exception.** The valid-id non-bounce path raises `VERPForward` (mapped to `E213`) and so is unaffected; only the *invalid-id* (`E512`) and *inactive-user* (`E510`) returns are candidates for suppression.

### G.5 Conclusion — reported exactly, not adjusted

Under SPF-fail/soft-fail the invalid case (`E216`, numeric `250`) and the valid non-bounce case (`E213`, numeric `250`) have the **same three-digit code**, so the *numeric* leak is masked. **But the SimpleLogin sub-code string still differs — `E216` vs `E213` — so the oracle survives at the string level.** Likewise the inactive-user `E510` (a `550`) is silenced to `E216` only under SPF-fail/soft-fail; under SPF-pass or an absent header it is preserved. This is reported as observed rather than adjusted toward a "fully mitigated" conclusion: the SPF rewrite narrows, but does not eliminate, the differential response.

---

## Phase H — Security framing (background context)

The observations above are grounded in captured status strings and `file:line` citations. The external references in this section frame those findings against established practice; each is a brief factual statement with a source link, not a substitute for the observed evidence.

### H.1 SMTP recipient-enumeration / differential-response disclosure

A mail server that returns materially different responses for existing vs. non-existing recipients or identifiers is a recognized information-disclosure weakness, catalogued by MITRE as [CWE-204: Observable Response Discrepancy](https://cwe.mitre.org/data/definitions/204.html) — a product that behaves differently in a way an unauthorized actor can observe, thereby revealing internal state (MITRE notes this "frequently" occurs in authentication, where differing responses reveal whether a credential is valid). The classic mail-layer analogue is SMTP `VRFY`/`EXPN` user enumeration, flagged for decades by scanners such as [Tenable/Nessus plugin 10249, "Multiple Mail Server EXPN/VRFY Information Disclosure"](https://www.tenable.com/plugins/nessus/10249), whose remediation guidance is to disable commands that hand an attacker "too much information." SimpleLogin's observed divergence maps directly onto CWE-204: `250 SL E213 Unknown email ignored` for a valid `email_log_id` vs. `550 SL E512 No such email log` for an invalid one — a textbook `250`-vs-`550` differential-response oracle, here over internal `email_log_id` integers rather than mailbox names.

### H.2 Bounce Address Tag Validation (BATV)

The signed format is a BATV-style scheme. BATV tags the envelope return-path with an unforgeable token so that backscatter/forged bounces can be rejected. The most-cited specification is the IETF Internet-Draft ["Bounce Address Tag Validation (BATV)", draft-levine-smtp-batv-01](https://datatracker.ietf.org/doc/html/draft-levine-smtp-batv-01) — **and its standing must be disclosed accurately: this is an EXPIRED Internet-Draft (it expired on 14 November 2008) and was never published as an RFC or adopted as an IETF standard.** It is cited here only as the historical description of the technique, not as a normative standard. Its `prvs` ("Simple Private Signature") variant tags `MAIL FROM` with a key id, a day-granularity expiry, and a keyed hash, and its own text acknowledges the short tag provides only weak replay protection. SimpleLogin's `generate_verp_email()`/`get_verp_info_from_email()` are a direct analogue: an HMAC-`sha3-224` (8-byte digest) over `[verp_type, object_id, minutes-since-VERP_TIME_START]`, gated by `VERP_MESSAGE_LIFETIME = 5 * 86400` (five days). The observed missing-lower-bound behavior (Phase F.2 — an old-dated `-6d` address is still accepted) is exactly the kind of replay-window laxity BATV's timestamp is meant to constrain.

### H.3 Variable Envelope Return Path (VERP)

VERP encodes per-message routing information into the envelope return-path so a bounce auto-identifies the failed recipient without parsing the human-readable bounce body; see [Postfix's VERP documentation](https://www.postfix.org/VERP_README.html) and the general concept overview at [Wikipedia: Variable envelope return path](https://en.wikipedia.org/wiki/Variable_envelope_return_path). Classic VERP exposes the identifier in **plaintext**; a known hazard is that implementations tend to assume any message arriving at a VERP bounce address *is* a bounce, so probes to that address can trigger bounce processing. This is precisely why SimpleLogin embeds an `email_log_id` in `bounce+{id}+@domain`, and why the OLD (plaintext) vs. NEW (signed) split exists: the OLD form is classic, forgeable VERP; the NEW form adds the BATV-style authentication tag.

**(inferred, framing only):** the report notes — but does not design or implement — that the signed format already closes the enumeration gap; remediation is out of scope for this investigation.

---

## Phase I — Status-string reference, exception mapping & full scenario matrix

### I.1 Verbatim status-string reference [app/email/status.py] — SOURCE-QUOTED vs RUNTIME-OBSERVED

The table quotes each constant verbatim from `app/email/status.py` and marks, honestly, whether this investigation **observed it at runtime** (with the phase/transcript that captured it) or merely **quotes it from source** (present in the code but not produced by any scenario exercised here).

| Const | Line | Literal string | Status in this investigation |
|-------|------|----------------|------------------------------|
| `E200` | L2 | `250 Message accepted for delivery` | **RUNTIME-OBSERVED** — iCloud signed fall-through (Phase C.5 / K.8, `obs_H.out`) |
| `E205` | L7 | `250 SL E205 bounce handled` | **RUNTIME-OBSERVED** — transactional bounce (Phase C.2 / K.6, `obs_E.out`) |
| `E206` | L9 | `250 SL E206 Out of office` | **RUNTIME-OBSERVED** — transactional OOO (Phase C.2 / K.6, `obs_E.out`) |
| `E211` | L19 | `250 SL E211 Bounce Forward phase handled` | **RUNTIME-OBSERVED** — forward bounce (Phase C.3 / K.3, `obs_B.out`) |
| `E212` | L20 | `250 SL E212 Bounce Reply phase handled` | **RUNTIME-OBSERVED** — reply bounce (Phase C.4 / K.3, `obs_B.out`) |
| `E213` | L21 | `250 SL E213 Unknown email ignored` | **RUNTIME-OBSERVED** — valid-id non-bounce (Phase D.1 / K.3, `obs_B.out`) |
| `E214` | L22 | `250 SL E214 Unauthorized for using reverse alias` | **RUNTIME-OBSERVED** — forward out-of-office fall-through, valid id (Phase C.3 / K.10, `gb_run*.out`) |
| `E216` | L24 | `250 SL E216 Handled spf policy` | **RUNTIME-OBSERVED** — SPF-fail rewrite (Phase G.2 / K.5, `obs_D.out`) |
| `E404` | L32 | `421 SL E404 Unexpected error - Retry later` | **SOURCE-QUOTED only** — the generic `except Exception` fallback [email_handler.py:L2332]; not triggered by any benign scenario here (note: `421`, not `550`) |
| `E502` | L39 | `550 SL E502 Email not exist` | **SOURCE-QUOTED only** — not on the bounce path exercised here |
| `E504` | L41 | `550 SL E504 Account disabled` | **SOURCE-QUOTED only** — not on the bounce path exercised here |
| `E510` | L47 | `550 SL E510 so such user` *(source typo preserved)* | **RUNTIME-OBSERVED** — inactive user (Phase D.3 / K.3, `obs_B.out`) |
| `E512` | L49 | `550 SL E512 No such email log` | **RUNTIME-OBSERVED** — invalid id (Phase D.1 / K.3, `obs_B.out`) |
| `E515` | L51 | `550 SL E515 Email not exist` | **RUNTIME-OBSERVED** — corrupted/future signed diversion (Phase F.3 / K.4, `obs_C.out`) |
| `E524` | L62 | `550 SL E524 Wrong use of reverse-alias` | **SOURCE-QUOTED only** — `except CannotCreateContactForReverseAlias` [email_handler.py:L2307]; not triggered here |

Note `E502` and `E515` carry identical text (`"Email not exist"`); the enumeration divergence is between `E512`/`E213`/`E510`, while the signature-failure diversion lands on `E515`.

### I.2 `handle_DATA()` exception mapping [email_handler.py:L2297-L2332]

Quoted complete (the three `except` clauses that shape the final wire response):

```python
        except CannotCreateContactForReverseAlias as e:
            LOG.w(
                "Probably due to reverse-alias used in the forward phase, "
                "error:%s mail_from:%s, rcpt_tos:%s, header_from:%s, header_to:%s",
                e,
                envelope.mail_from,
                envelope.rcpt_tos,
                msg[headers.FROM],
                msg[headers.TO],
            )
            return status.E524
        except (VERPReply, VERPForward, VERPTransactional) as e:
            LOG.w(
                "email handling fail with error:%s "
                "mail_from:%s, rcpt_tos:%s, header_from:%s, header_to:%s",
                e,
                envelope.mail_from,
                envelope.rcpt_tos,
                msg[headers.FROM],
                msg[headers.TO],
            )
            return status.E213
        except Exception as e:
            LOG.e(
                "email handling fail with error:%s "
                "mail_from:%s, rcpt_tos:%s, header_from:%s, header_to:%s, saved to %s",
                e,
                envelope.mail_from,
                envelope.rcpt_tos,
                msg[headers.FROM],
                msg[headers.TO],
                save_envelope_for_debugging(
                    envelope, file_name_prefix=e.__class__.__name__
                ),  # todo: remove
            )
            return status.E404
```

The three `VERP*` exception classes are defined in `app/errors.py` (`VERPTransactional` [app/errors.py:L42-L44], `VERPForward` [app/errors.py:L48-L50], `VERPReply` [app/errors.py:L54-L56]); each is mapped to `E213` at [email_handler.py:L2318]. This is why every valid-id **non-bounce** probe surfaces as `250 SL E213 Unknown email ignored` at the `handle_DATA()` layer.

### I.3 Full scenario matrix (one row per distinct condition; no merged verdicts)

Legend — layers: `H` = `handle()`, `_h` = `_handle()`, `DATA` = `handle_DATA()`. "STABLE" = identical outcome across the two runs recorded for each script (Phase K). SPF is `allow`/`absent` unless a specific verdict is named. Every "Result" is the complete literal string.

| # | Address format | id | `is_bounce` | SPF | Layer(s) | Result (literal) | STABLE | Evidence |
|---|----------------|----|-----------|-----|----------|------------------|--------|----------|
| 1 | OLD unsigned `bounce+99999999999999+@sl.local` | invalid | F | allow/absent | H, _h, DATA | `550 SL E512 No such email log` | yes | K.3 `obs_B.out` |
| 2 | OLD unsigned `bounce+385+@sl.local` | valid | F | allow/absent | H, _h | RAISE `VERPForward` | yes | K.3 `obs_B.out` |
| 3 | OLD unsigned `bounce+385+@sl.local` | valid | F | allow/absent | DATA | `250 SL E213 Unknown email ignored` | yes | K.3 `obs_B.out` |
| 4 | OLD unsigned `bounce+99999999999999+@sl.local` | invalid | T | allow/absent | H, _h, DATA | `550 SL E512 No such email log` | yes | K.3 `obs_B.out` |
| 5 | OLD unsigned `bounce+386+@sl.local` | valid (active) | T | allow/absent | H, _h, DATA | `250 SL E211 Bounce Forward phase handled` | yes | K.3 `obs_B.out` |
| 6 | NEW signed forward (valid, inactive user) | valid (inactive) | T | allow/absent | H, _h, DATA | `550 SL E510 so such user` | yes | K.3 `obs_B.out` |
| 7 | NEW signed forward | invalid | F | allow/absent | H, _h, DATA | `550 SL E512 No such email log` | yes | K.3 `obs_B.out` |
| 8 | NEW signed forward | valid | F | allow/absent | H, _h | RAISE `VERPForward` | yes | K.3 `obs_B.out` |
| 9 | NEW signed forward | valid | F | allow/absent | DATA | `250 SL E213 Unknown email ignored` | yes | K.3 `obs_B.out` |
| 10 | OLD unsigned reply `bounce_reply+99999999999999+@sl.local` | invalid | T | allow/absent | H, _h, DATA | `550 SL E512 No such email log` | yes | K.3 `obs_B.out` |
| 11 | NEW signed reply | valid | T | allow/absent | H, _h, DATA | `250 SL E212 Bounce Reply phase handled` | yes | K.3 `obs_B.out` |
| 12 | NEW signed reply | valid | F | allow/absent | H -> DATA | RAISE `VERPReply` -> `250 SL E213 Unknown email ignored` | yes | K.3 `obs_B.out` |
| 13 | NEW signed reply | invalid | F | allow/absent | H, _h, DATA | `550 SL E512 No such email log` | yes | K.3 `obs_B.out` |
| 14 | Signed transactional | invalid | F | allow/absent | H -> DATA | RAISE `VERPTransactional` -> `250 SL E213 Unknown email ignored` | yes | K.6 `obs_E.out` |
| 15 | Signed transactional | invalid | T | allow/absent | H, DATA | `250 SL E205 bounce handled` (no `E512`) | yes | K.6 `obs_E.out` |
| 16 | Signed transactional | invalid | F (Auto-Submitted OOO) | allow/absent | H, DATA | `250 SL E206 Out of office` | yes | K.6 `obs_E.out` |
| 17 | iCloud: OLD in `mail_from` `bounce+397+@sl.local` | valid | (no gate) | allow/absent | H | `250 SL E211 Bounce Forward phase handled` | yes | K.6 `obs_E.out` (direct `handle()`, non-canonical; canonical via `handle_DATA` = row 47) |
| 18 | iCloud: OLD in `mail_from` `bounce+99999999999999+@sl.local` | invalid | (no gate) | allow/absent | H | `550 SL E512 No such email log` | yes | K.6 `obs_E.out` (direct `handle()`, non-canonical; canonical via `handle_DATA` = row 48) |
| 19 | iCloud: NEW signed in `mail_from` | valid | (no gate) | allow/absent | DATA | `250 Message accepted for delivery` (fall-through, `mail_from[0]` bug) | yes | K.8 `obs_H.out` |
| 20 | SIGNED GOOD (control) | valid `392` | F | allow/absent | H -> DATA | RAISE `VERPForward` -> `250 SL E213 Unknown email ignored` | yes | K.4 `obs_C.out` |
| 21 | SIGNED CORRUPTED (bad HMAC) | — | F | allow/absent | H, DATA | `550 SL E515 Email not exist` | yes | K.4 `obs_C.out` |
| 22 | SIGNED FUTURE-dated `+6d` | valid `392` | F | allow/absent | H, DATA | `550 SL E515 Email not exist` | yes | K.4 `obs_C.out` |
| 23 | SIGNED OLD-dated `-6d` | valid `392` | F | allow/absent | H -> DATA | RAISE `VERPForward` -> `250 SL E213 Unknown email ignored` (accepted) | yes | K.4 `obs_C.out` |
| 24 | OLD unsigned | invalid | F | **fail** | _h | `250 SL E216 Handled spf policy` | yes | K.5 `obs_D.out` |
| 25 | OLD unsigned | invalid | F | **soft-fail** | _h | `250 SL E216 Handled spf policy` | yes | K.5 `obs_D.out` |
| 26 | OLD unsigned | invalid | F | **allow** | _h | `550 SL E512 No such email log` | yes | K.5 `obs_D.out` |
| 27 | OLD unsigned | invalid | F | **absent** | _h | `550 SL E512 No such email log` | yes | K.5 `obs_D.out` |
| 28 | NEW signed forward | invalid | F | **fail** | _h | `250 SL E216 Handled spf policy` | yes | K.5 `obs_D.out` |
| 29 | NEW signed forward | invalid | F | **soft-fail** | _h | `250 SL E216 Handled spf policy` | yes | K.5 `obs_D.out` |
| 30 | NEW signed forward | invalid | F | **allow** | _h | `550 SL E512 No such email log` | yes | K.5 `obs_D.out` |
| 31 | NEW signed forward | invalid | F | **absent** | _h | `550 SL E512 No such email log` | yes | K.5 `obs_D.out` |
| 32 | OLD unsigned | valid (inactive) `393` | T | **fail** | _h | `250 SL E216 Handled spf policy` (`E510` silenced) | yes | K.5 `obs_D.out` |
| 33 | OLD unsigned | valid (inactive) `393` | T | **allow** | _h | `550 SL E510 so such user` (preserved) | yes | K.5 `obs_D.out` |
| 34 | OLD unsigned | valid (inactive) `393` | T | **absent** | _h | `550 SL E510 so such user` (preserved) | yes | K.5 `obs_D.out` |
| 35 | OLD unsigned | invalid | F | **fail** | DATA | `250 SL E216 Handled spf policy` | yes | K.5 `obs_D2.out` |
| 36 | OLD unsigned | valid `395` | F | **fail** | DATA | `250 SL E213 Unknown email ignored` (never suppressed) | yes | K.5 `obs_D2.out` |
| 37 | OLD unsigned | valid `395` | F | **allow** | DATA | `250 SL E213 Unknown email ignored` | yes | K.5 `obs_D2.out` |
| 38 | OLD unsigned | valid `395` | F | **absent** | DATA | `250 SL E213 Unknown email ignored` | yes | K.5 `obs_D2.out` |
| 39 | OLD unsigned forward `bounce+349+@sl.local` | valid | F (Auto-Submitted OOO) | allow/absent | DATA | `250 SL E214 Unauthorized for using reverse alias` | yes | K.10 `gb_run*.out` |
| 40 | OLD unsigned forward `bounce+99999999999999+@sl.local` | invalid | F (Auto-Submitted OOO) | allow/absent | DATA | `550 SL E512 No such email log` | yes | K.10 `gb_run*.out` |
| 41 | NEW signed forward | valid | F (Auto-Submitted OOO) | allow/absent | DATA | `250 SL E214 Unauthorized for using reverse alias` | yes | K.10 `gb_run*.out` |
| 42 | NEW signed forward | invalid | F (Auto-Submitted OOO) | allow/absent | DATA | `550 SL E512 No such email log` | yes | K.10 `gb_run*.out` |
| 43 | OLD unsigned reply `bounce_reply+350+@sl.local` | valid | F (Auto-Submitted OOO) | allow/absent | DATA | `250 Message accepted for delivery` | yes | K.10 `gb_run*.out` |
| 44 | OLD unsigned reply `bounce_reply+99999999999999+@sl.local` | invalid | F (Auto-Submitted OOO) | allow/absent | DATA | `550 SL E512 No such email log` | yes | K.10 `gb_run*.out` |
| 45 | NEW signed reply | valid | F (Auto-Submitted OOO) | allow/absent | DATA | `250 Message accepted for delivery` | yes | K.10 `gb_run*.out` |
| 46 | NEW signed reply | invalid | F (Auto-Submitted OOO) | allow/absent | DATA | `550 SL E512 No such email log` | yes | K.10 `gb_run*.out` |
| 47 | iCloud: OLD in `mail_from` `bounce+353+@sl.local` (canonical `handle_DATA`) | valid | (no gate) | allow/absent | DATA | `250 SL E211 Bounce Forward phase handled` | yes | K.10 `gb_run*.out` |
| 48 | iCloud: OLD in `mail_from` `bounce+99999999999999+@sl.local` (canonical `handle_DATA`) | invalid | (no gate) | allow/absent | DATA | `550 SL E512 No such email log` | yes | K.10 `gb_run*.out` |

**Determinism spot-check (exact-input repetition):** rows 1/7 (invalid-id `_handle()`) and row 3 (valid-id `handle_DATA()`) were each additionally repeated **5×** on the same unchanged input with 5/5 identical results (Phase A.6 / K.9, `obs_G.out`).

Auxiliary observations (helper-level, **non-canonical** because they bypass the handler chain):

- **(non-canonical, direct call)** `parse_id_from_bounce('bounce+12345+@sl.local')` = `12345` (Phase B.1 / K.2).
- **(non-canonical, direct call)** `get_verp_info_from_email()`: GOOD -> `(<VerpType.bounce_forward: 0>, 392)`; CORRUPTED -> `None`; `+6d` -> `None`; `-6d` -> `(<VerpType.bounce_forward: 0>, 392)` (Phase F.3 / K.4).
- **(non-canonical, direct call)** `is_bounce` truth table (Phase E.2 / K.6).

---

## Phase I.4 — Final coverage pass (marked complete only where runtime evidence exists)

Each objective, and each named mechanism/condition within it, cross-checked against captured evidence. `[x]` means observed at runtime with an embedded transcript; where something is source-quoted rather than observed, it is labeled as such.

- [x] **Objective 1 — the two address formats.** OLD unsigned `bounce+{id}+@sl.local` (`BOUNCE_PREFIX` [app/config.py:L100], `BOUNCE_SUFFIX` [app/config.py:L101]; `parse_id_from_bounce` [app/email_utils.py:L1258-L1259], observed `parse_id_from_bounce('bounce+12345+@sl.local')=12345`) vs NEW signed via `generate_verp_email` [app/email_utils.py:L1438-L1464] / `get_verp_info_from_email` [app/email_utils.py:L1467-L1498], observed round-trips for `bounce_forward`/`bounce_reply`/`transactional` (id `369`). Terminology corrected to **signed/authenticated cleartext** (not encrypted). — **Phase B**, transcript **K.2** (`obs_A.out`).
- [x] **Objective 2 — routing / four branches.** Chain `handle_DATA` [email_handler.py:L2289] -> `_handle` [email_handler.py:L2335] -> `handle` [email_handler.py:L1945]; routing block [email_handler.py:L2034-L2116]; transactional [email_handler.py:L2038-L2054] (observed `E205`/`E206`/`VERPTransactional`->`E213`), forward-bounce [email_handler.py:L2057-L2074] (observed `E512`/`VERPForward`->`E213`/`E211`, and the **out-of-office fall-through** `250 SL E214 Unauthorized for using reverse alias` for a valid id — canonically via `handle_DATA`, Phase C.3), reply-bounce [email_handler.py:L2077-L2098] (observed `E512`/`VERPReply`->`E213`/`E212`, and the OOO fall-through `250 Message accepted for delivery` (`E200`) for a valid id — canonically via `handle_DATA`, Phase C.4), iCloud [email_handler.py:L2100-L2116] (observed **canonically via `handle_DATA`**: valid unsigned `E211`, invalid `E512` — Phase C.5, I.3 rows 47-48). — **Phase C**, transcripts **K.3, K.8, K.10** (canonical OOO / iCloud / state), with the non-canonical direct-`handle()`/`is_bounce()` illustrations labelled as such in **K.6**.
- [x] **Objective 3 — external-attacker feasibility.** Only the envelope recipient + a body are required; `550 SL E512 No such email log` (invalid id `99999999999999`) vs `250 SL E213 Unknown email ignored` (valid id `385`) observed via the canonical chain. — **Phase D.1**, transcript **K.3**.
- [x] **Objective 4 — the security boundary.** Unsigned -> `parse_id_from_bounce` -> `EmailLog.get` [email_handler.py:L2063] with no crypto; signed -> `get_verp_info_from_email` returns `None` on **six** conditions (each enumerated with `file:line` in Phase F.1) — including HMAC mismatch [app/email_utils.py:L1490-L1491] and future-expiry [app/email_utils.py:L1496]; corrupted-HMAC and future-dated signed addresses observed to divert to `550 SL E515 Email not exist` [app/email/status.py:L51]; **one-sided expiry** confirmed (old `-6d` still accepted -> `250 SL E213 Unknown email ignored`). The FUTURE `+6d` / OLD `-6d` boundary *input* addresses were minted by the disclosed **signer-clock shim** (`patch("app.email_utils.time")`, scoped to address generation only — Phases A.2 and K.4); the verifier and handler ran **unpatched**. The four attacker-reachable verifier shapes (conditions 1-4: no `@`, wrong prefix / short field list, non-base32 payload, corrupted HMAC) were additionally driven **canonically** through `handle_DATA` (all -> `E515`). — **Phase F**, transcripts **K.4** (GOOD/CORRUPTED/FUTURE/OLD) and **K.11** (verifier-shape).
- [x] **Objective 5 — exact SMTP responses.** RUNTIME-OBSERVED verbatim: `E200`, `E205`, `E206`, `E211`, `E212`, `E213`, `E216`, `E510`, `E512`, `E515` (see I.1 for the phase/transcript of each). SOURCE-QUOTED only (present in code, not triggered here): `E404` (`421 SL E404 Unexpected error - Retry later`), `E502`, `E504`, `E524`. — **Phases C/D/F/G/I**, transcripts **K.3–K.8**.
- [x] **Objective 6 — `is_bounce` and spoofability.** `is_bounce` [email_handler.py:L1813-L1818] reads only `envelope.mail_from == "<>"` and `msg.get_content_type().lower() == "multipart/report"`, both attacker-controllable; truth table observed (non-canonical direct calls); canonical pass/fail effect observed. — **Phase E**, transcript **K.6**.
- [x] **Objective 7 — richness + ordering.** Inactive-user `550 SL E510 so such user` [handle_bounce, email_handler.py:L1869-L1871]; existence check `E512` [email_handler.py:L2063-L2067] precedes the `is_bounce` gate [email_handler.py:L2069], so a non-bounce probe still enumerates; committed state transition (`bounced`/`refused_email_id`/`bounced_mailbox_id`, `+1` Bounce/`+1` RefusedEmail) captured **canonically via `handle_DATA`** with BEFORE/AFTER values across two runs (Phase D.4). — **Phase D.2–D.4**, transcripts **K.3** and **K.10** (canonical `handle_DATA` state transition); the non-canonical direct-`handle()` version is labelled as such in **K.7**.

**Named-item coverage (each addressed by name):** `handle` [email_handler.py:L1945], `_handle` [email_handler.py:L2335], `handle_DATA` [email_handler.py:L2289], `is_bounce` [email_handler.py:L1813-L1818], `handle_bounce` [email_handler.py:L1851], `handle_bounce_forward_phase` [email_handler.py:L1432-L1520], `is_automatic_out_of_office` [email_handler.py:L1793-L1812], `parse_id_from_bounce` [app/email_utils.py:L1258-L1259], `generate_verp_email` [app/email_utils.py:L1438-L1464], `get_verp_info_from_email` [app/email_utils.py:L1467-L1498], `EmailLog.get` [email_handler.py:L2063], `User.is_active` [app/models.py:L766-L769], `VerpType` [app/models.py:L247-L250], `SpamdResult.extract_from_headers` [app/handler/spamd_result.py:L76] / `SPFCheckResult` [app/handler/spamd_result.py:L32-L46], `VERPForward`/`VERPReply`/`VERPTransactional` [app/errors.py:L42-L56]; status codes `E200` [app/email/status.py:L2], `E205` [app/email/status.py:L7], `E206` [app/email/status.py:L9], `E211` [app/email/status.py:L19], `E212` [app/email/status.py:L20], `E213` [app/email/status.py:L21], `E214` [app/email/status.py:L22], `E216` [app/email/status.py:L24], `E404` [app/email/status.py:L32], `E502` [app/email/status.py:L39], `E504` [app/email/status.py:L41], `E510` [app/email/status.py:L47], `E512` [app/email/status.py:L49], `E515` [app/email/status.py:L51], `E524` [app/email/status.py:L62]; config `BOUNCE_PREFIX` [app/config.py:L100], `BOUNCE_SUFFIX` [app/config.py:L101], `BOUNCE_PREFIX_FOR_REPLY_PHASE` [app/config.py:L108-L110], `TRANSACTIONAL_BOUNCE_PREFIX` [app/config.py:L113-L114], `VERP_PREFIX` [app/config.py:L500], `VERP_MESSAGE_LIFETIME` [app/config.py:L499], `VERP_EMAIL_SECRET` [app/config.py:L502]; constants `VERP_TIME_START`/`VERP_HMAC_ALGO` [app/email_utils.py:L68-L69].

### I.5 Methodology attestation

- All behavioral claims are backed by a verbatim captured status string (with the producing layer and the transcript in Phase K) or a `file:line` citation; interpretations are labeled **(inferred)**.
- **No observed value was produced under a mock.** The verifier `get_verp_info_from_email()`, the routing `handle()` / `_handle()` / `handle_DATA()`, the PostgreSQL database, and the SMTP response path all ran **unmocked** for every captured status string and every state transition. The **single** use of `unittest.mock.patch` across all observation scripts is a **signer-clock shim** (`patch("app.email_utils.time")`) in Script C, used **only** to mint the two timestamp-boundary *input* addresses (FUTURE `+6d`, OLD `-6d`); it patches nothing in the verifier or the handler chain and is released before submission (fully disclosed in Phases A.2 and K.4).
- **Canonical-path fidelity is labelled at every use.** Any value obtained by calling a helper directly (`parse_id_from_bounce`, `is_bounce`, `get_verp_info_from_email`, or the module-level `handle()` invoked without the `handle_DATA` / `_handle` wrapper) is explicitly marked **(non-canonical, direct call)** at the point of use; the iCloud branch and the committed state transition are **additionally** driven through the real `handle_DATA` entry point (Phases C.5, D.4; transcript K.10), and the verifier-rejection shapes through `handle_DATA` (Phase K.11).
- **Run-to-run determinism is backed by an executable comparison,** not asserted in prose: the ordered-outcome sequences of the repeated runs are extracted and compared by a small program whose command and complete real output — together with a complete second-run transcript — are embedded in Phase K.0 (canonical matrix `sha256=6d440cc5…` across four runs; Group-B script `sha256=f30651b8…` across three runs).
- The direct answer leads; nuances (one-sided expiry; SPF numeric convergence with string-level survival) follow; no observed value was adjusted toward an expected answer.
- The mandated-vs-actual environment difference is disclosed (Phase A.3), not normalized: the in-image datastore is **PostgreSQL 15.13**, but the AAP-mandated **PostgreSQL 13** was additionally provisioned (**13.23**) and the **entire canonical matrix re-run against it twice** — byte-identical to PostgreSQL 15 (complete transcripts + executable comparison in **Phase K.1a**), so the mandated major version is now backed by real captured evidence rather than left as an open discrepancy; `aiosmtpd 1.4.6` was mandated but the `poetry.lock`-pinned **1.4.2** is the actual, and (being off the status-string path) affects no response.
- Database state changes are reported as **committed mutations to the ephemeral test DB** (SQLAlchemy 1.3.24 does not roll them back in this harness), not as transient/rolled-back changes (Phase A.5).
- The user's example `bounce+12345+@domain` is reproduced as `bounce+12345+@sl.local` with `parse_id_from_bounce('bounce+12345+@sl.local') = 12345`.
- No remediation is proposed or implemented (out of scope); it is noted only, and labeled, that the signed format already closes the gap.

---

## Phase J — Validation & repository cleanliness

This phase records the exact commands used to remove every temporary artifact and to prove that the source repository is left byte-for-byte unchanged apart from this single document.

### J.1 Temporary-artifact cleanup

All observation scripts and transcripts lived **outside** the repository — in the container at `/tmp/blitzy_evidence/` (a non-bind-mounted overlay) and mirrored on the host at `/tmp/blitzy_evidence_host/`, plus the document staging directory `/tmp/doc_build/`. None of these paths is inside the repository checkout, so none could ever appear in `git status`. They are removed with:

```text
# container-side temporary scripts + transcripts (non-bind-mounted overlay):
docker exec sl-app-0 rm -rf /tmp/blitzy_evidence
# host-side backup + document staging (both outside the repo checkout):
rm -rf /tmp/blitzy_evidence_host /tmp/doc_build
```

This investigation created no `blitzy_adhoc_test_*` files, build artifacts, or virtualenvs inside the repository checkout — every temporary observation script lived **outside** it (Phase K), so none could appear in `git status`. The git-tracked repository is left byte-for-byte unchanged apart from this single document. (Python bytecode caches under `__pycache__/` are gitignored by `.gitignore`'s `*.pyc` rule and are excluded from git tracking; any such cache left in the working tree by a setup- or platform-phase `pytest` collection — for example a `tests/__pycache__/*.pyc` — never appears in `git status`/`git diff` and does not affect this byte-for-byte guarantee.)

### J.2 Repository-cleanliness verification

Before committing, the working tree contains exactly one modified path — this document — and nothing else:

```text
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --name-status
M	blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --check
(no output -- zero whitespace/EOF errors)
```

`git status --porcelain` lists exactly one path -- this document; `git diff --name-status` confirms it is the single modified (`M`) file and nothing else changed; and `git diff --check` returns no output, confirming there are no trailing-whitespace or blank-EOF issues (the document ends with a single newline after its last non-blank line).

### J.3 Reference-file integrity

Every source and test file consulted in this investigation — `email_handler.py`, `app/email_utils.py`, `app/config.py`, `app/email/status.py`, `app/errors.py`, `app/models.py`, `app/handler/spamd_result.py`, `tests/test_email_handler.py`, `tests/utils.py`, `tests/conftest.py`, `tests/test.env`, `tests/example_emls/5xx_overwrite_spf.eml`, `tests/example_emls/no_spamd_header.eml` — was used **read-only** (consulted and executed, never edited). They do not appear in `git status`, confirming they are unchanged. The only tracked change in the repository is `blitzy/documentation/app_2cd6ee777f8c.md`.

---

## Phase K — Evidence appendix: complete verbatim transcripts & observation-script sources

Per the evidence rule, this appendix embeds, for every observation, (a) the exact command, (b) the complete source of the temporary observation script, and (c) the complete stdout+stderr transcript, each reproduced verbatim. The **only** normalization applied is that trailing whitespace at line-ends has been trimmed for repository cleanliness (so `git diff --check` is clean, Phase J): no line, no character of content, no status string, no value, and no log message is removed, reordered, summarized, or otherwise altered. Nothing is elided. Every script was invoked with the form:

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/<script>.py
```

The SL log lines carry `/work/...` paths because the repository is bind-mounted at `/work` inside the container `sl-app-0`; `/work` is the same checkout as the host repository root. All scripts and transcripts lived under `/tmp/blitzy_evidence/` (container) and `/tmp/blitzy_evidence_host/` (host), both **outside** the repository, and were removed per Phase J.

### K.0 Two-run stability verification

Every canonical observation script was executed **at least twice** and the ordered sequence of outcomes (the quoted SL status strings) is identical between runs; only volatile tokens (random fixture names, per-run message-ids, the minute-bucketed base32 signature, and sequence-backed row ids) differ. This is proven by an **executable comparison program** whose complete real output is embedded below, computed over the complete run transcripts embedded verbatim elsewhere in this appendix: the canonical oracle matrix `obs_matrix.py` (PG15 run1 & run2, plus PG13 run1 & run2) in Phase **K.1a**, and the out-of-office/iCloud/state script `obs_group_b.py` (run1, run2 & run3) in Phase **K.10**.

**Executable comparison program (`compare_all_runs.py`):**

```python
#!/usr/bin/env python3
"""Executable multi-run stability comparison for the canonical observation scripts.
Extraction method matches Phase K.1a's compare_runs.py exactly: from each transcript, take the
ordered outcome lines (those containing '=>') that sit inside the script's OUTCOME SUMMARY block,
join them, and SHA-256 the result. Volatile tokens (fixture names, message-ids, base32 signatures,
sequence-backed ids) are OUTSIDE the summary block and thus excluded; only the observable OUTCOMES
are compared."""
import hashlib, sys

def outcomes(path, start_marker):
    out, inb = [], False
    for line in open(path):
        if start_marker in line:
            inb = True; continue
        if "END SUMMARY" in line:
            inb = False; continue
        if inb and "=>" in line:
            out.append(line.rstrip("\n"))
    return out

def compare(label, runs, start_marker):
    print("### %s" % label)
    hs = []
    for name, path in runs:
        oc = outcomes(path, start_marker)
        h = hashlib.sha256("\n".join(oc).encode()).hexdigest()
        hs.append(h)
        print("  %-14s %2d outcomes  sha256=%s" % (name, len(oc), h))
    ident = len(set(hs)) == 1
    print("  => %s across %d runs (%s)\n" % (
        "ALL RUNS BYTE-IDENTICAL" if ident else "DIFFER", len(runs),
        "STABLE" if ident else "UNSTABLE"))
    return ident

ok = True
ok &= compare(
    "obs_matrix.py — canonical oracle matrix (fwd/reply/signed/inactive/transactional/iCloud)",
    [("PG15 run1", "pg15_run1.out"), ("PG15 run2", "pg15_run2.out"),
     ("PG13 run1", "pg13_run1.out"), ("PG13 run2", "pg13_run2.out")],
    "CANONICAL OUTCOME SUMMARY")
ok &= compare(
    "obs_group_b.py — OUT-OF-OFFICE matrix + iCloud + state transition",
    [("PG15 run1", "gb_run1.out"), ("PG15 run2", "gb_run2.out"), ("PG15 run3", "gb_run3.out")],
    "GROUP-B OUTCOME SUMMARY")
print("OVERALL: %s" % ("ALL CANONICAL SCRIPTS STABLE ACROSS ALL RUNS" if ok else "INSTABILITY DETECTED"))
sys.exit(0 if ok else 1)
```

**Command and complete real output:**

```text
$ python3 compare_all_runs.py
### obs_matrix.py — canonical oracle matrix (fwd/reply/signed/inactive/transactional/iCloud)
  PG15 run1      14 outcomes  sha256=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354
  PG15 run2      14 outcomes  sha256=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354
  PG13 run1      14 outcomes  sha256=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354
  PG13 run2      14 outcomes  sha256=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354
  => ALL RUNS BYTE-IDENTICAL across 4 runs (STABLE)

### obs_group_b.py — OUT-OF-OFFICE matrix + iCloud + state transition
  PG15 run1      12 outcomes  sha256=f30651b8bf0ed96c03d81e4f4db16a396d259ffcaf69901d5932bb6abd4a4889
  PG15 run2      12 outcomes  sha256=f30651b8bf0ed96c03d81e4f4db16a396d259ffcaf69901d5932bb6abd4a4889
  PG15 run3      12 outcomes  sha256=f30651b8bf0ed96c03d81e4f4db16a396d259ffcaf69901d5932bb6abd4a4889
  => ALL RUNS BYTE-IDENTICAL across 3 runs (STABLE)

OVERALL: ALL CANONICAL SCRIPTS STABLE ACROSS ALL RUNS
```

=> The canonical oracle matrix (`obs_matrix.py`) is byte-identical across **4 runs** (two PostgreSQL 15 runs and two PostgreSQL 13 runs; `sha256=6d440cc5…` — the same hash reported by the independent `compare_runs.py` in Phase K.1a), and the out-of-office/iCloud/state script (`obs_group_b.py`) is byte-identical across **3 runs** (`sha256=f30651b8…`). The `obs_group_b.py` outcome rows are additionally verified in Phase K.10 by a full-summary-block SHA (`fc1c4be7…`); the two hashes differ only because K.0/K.1a hash the ordered `=>` outcome lines while K.10 hashes the same rows **including** the block header and end marker — both are reproducible over the identical embedded transcripts.

**Complete second-run transcript — canonical oracle matrix, PostgreSQL 15 run 2 (`pg15_run2.out`)** (the run-1 counterpart and the PG13 run1/run2 pair are embedded in Phase K.1a; the `obs_matrix.py` source is in Phase K.1a):

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/tjnikxnuyvdwhmzbnbpv
Upload files to local dir
>>> init logging <<<
2026-07-14 02:12:05,943 - SL - DEBUG - 18626 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-14 02:12:06,994 - SL - DEBUG - 18626 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-14 02:12:06,995 - SL - DEBUG - 18626 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-14 02:12:06,996 - SL - DEBUG - 18626 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 02:12:06,997 - SL - DEBUG - 18626 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
=== DB ENV FACTS ===
DB_URI(host:port/db) = localhost:15432/test
SELECT version()     = PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
alembic head         = 32f25cbf12f6
public table count   = 77
python               = 3.10.18
aiosmtpd             = 1.4.2
EMAIL_DOMAIN         = sl.local
=== END ENV FACTS ===

2026-07-14 02:12:07,310 - SL - INFO - 18626 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:12:07,326 - SL - DEBUG - 18626 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email erring_colder311@sl.local
2026-07-14 02:12:07,335 - SL - INFO - 18626 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURES user.id=453 alias=erring_colder311@sl.local valid_fwd=(344,345) valid_reply=346 INVALID=99999999999999

2026-07-14 02:12:07,374 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() -  - set message_id 2967f8f6-f0fd-498c-a294-a33b0f488dae
2026-07-14 02:12:07,374 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - 2967f8f6-f0fd-498c-a294-a33b0f488dae - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,375 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - 2967f8f6-f0fd-498c-a294-a33b0f488dae - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-14 02:12:07,375 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - 2967f8f6-f0fd-498c-a294-a33b0f488dae - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,375 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - 2967f8f6-f0fd-498c-a294-a33b0f488dae - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,377 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - 2967f8f6-f0fd-498c-a294-a33b0f488dae - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,382 - SL - WARNING - 18626 - "/work/email_handler.py:2066" - handle() - 2967f8f6-f0fd-498c-a294-a33b0f488dae - No such email log
2026-07-14 02:12:07,382 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - 2967f8f6-f0fd-498c-a294-a33b0f488dae - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.007331132888793945 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD OLD  INVALID non-bounce    mail_from='attacker@evil.example' rcpt='bounce+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:12:07,383 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - 2967f8f6-f0fd-498c-a294-a33b0f488dae - set message_id a3a87c68-81de-4a43-b31f-e89092e3db5a
2026-07-14 02:12:07,383 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - a3a87c68-81de-4a43-b31f-e89092e3db5a - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,383 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - a3a87c68-81de-4a43-b31f-e89092e3db5a - New message, mail from attacker@evil.example, rctp tos ['bounce+344+@sl.local']
2026-07-14 02:12:07,383 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - a3a87c68-81de-4a43-b31f-e89092e3db5a - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,383 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - a3a87c68-81de-4a43-b31f-e89092e3db5a - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,384 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - a3a87c68-81de-4a43-b31f-e89092e3db5a - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+344+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,388 - SL - WARNING - 18626 - "/work/email_handler.py:2309" - handle_DATA() - a3a87c68-81de-4a43-b31f-e89092e3db5a - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+344+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL FWD OLD  VALID   non-bounce    mail_from='attacker@evil.example' rcpt='bounce+344+@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:12:07,388 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - a3a87c68-81de-4a43-b31f-e89092e3db5a - set message_id a601ede3-5564-4ef4-959a-3e769397114f
2026-07-14 02:12:07,388 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - a601ede3-5564-4ef4-959a-3e769397114f - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,388 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - a601ede3-5564-4ef4-959a-3e769397114f - New message, mail from <>, rctp tos ['bounce+345+@sl.local']
2026-07-14 02:12:07,389 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - a601ede3-5564-4ef4-959a-3e769397114f - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,389 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - a601ede3-5564-4ef4-959a-3e769397114f - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,390 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - a601ede3-5564-4ef4-959a-3e769397114f - ==>> Handle mail_from:<>, rcpt_tos:['bounce+345+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,398 - SL - DEBUG - 18626 - "/work/email_handler.py:1862" - handle_bounce() - a601ede3-5564-4ef4-959a-3e769397114f - handle bounce for <EmailLog 345>, phase=forward, contact=<Contact 155 contact@example.com 759>, alias=<Alias 759 erring_colder311@sl.local>
2026-07-14 02:12:07,400 - SL - ERROR - 18626 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - a601ede3-5564-4ef4-959a-3e769397114f - Use <Alias 759 erring_colder311@sl.local> default mailbox <Mailbox 529 user_ie2z24kpww@mailbox.test>
NoneType: None
2026-07-14 02:12:07,400 - SL - WARNING - 18626 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - a601ede3-5564-4ef4-959a-3e769397114f - cannot get bounce info, debug at
2026-07-14 02:12:07,403 - SL - DEBUG - 18626 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - a601ede3-5564-4ef4-959a-3e769397114f - Handle forward bounce <Contact 155 contact@example.com 759> -> <Alias 759 erring_colder311@sl.local> -> <Mailbox 529 user_ie2z24kpww@mailbox.test>. <EmailLog 345>
2026-07-14 02:12:07,407 - SL - WARNING - 18626 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - a601ede3-5564-4ef4-959a-3e769397114f - Cannot parse original message from bounce message <Alias 759 erring_colder311@sl.local> <User 453 Test User user_ie2z24kpww@mailbox.test> <Contact 155 contact@example.com 759> refused-emails/full-054ef1c3-ff73-4c9c-8b18-ea0882f8888b.eml
2026-07-14 02:12:07,410 - SL - DEBUG - 18626 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - a601ede3-5564-4ef4-959a-3e769397114f - Create refused email <Refused Email 114 None 2026-07-21T02:12:07.409892+00:00>
2026-07-14 02:12:07,423 - SL - DEBUG - 18626 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - a601ede3-5564-4ef4-959a-3e769397114f - Inform user <User 453 Test User user_ie2z24kpww@mailbox.test> about a bounce from contact <Contact 155 contact@example.com 759> to alias <Alias 759 erring_colder311@sl.local>
2026-07-14 02:12:07,454 - SL - DEBUG - 18626 - "/work/app/email_utils.py:303" - send_email() - a601ede3-5564-4ef4-959a-3e769397114f - send email to user_ie2z24kpww@mailbox.test, subject 'An email sent to erring_colder311@sl.local cannot be delivered to your mailbox'
2026-07-14 02:12:07,458 - SL - DEBUG - 18626 - "/work/app/mail_sender.py:131" - send() - a601ede3-5564-4ef4-959a-3e769397114f - send email with subject 'An email sent to erring_colder311@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_ie2z24kpww@mailbox.test'
2026-07-14 02:12:07,459 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - a601ede3-5564-4ef4-959a-3e769397114f - Finish mail_from <>, rcpt_tos ['bounce+345+@sl.local'], takes 0.07018256187438965 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
CELL FWD OLD  VALID   bounce        mail_from='<>'                   rcpt='bounce+345+@sl.local'
     handle_DATA() = '250 SL E211 Bounce Forward phase handled'
2026-07-14 02:12:07,459 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - a601ede3-5564-4ef4-959a-3e769397114f - set message_id 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2
2026-07-14 02:12:07,459 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,459 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgmzf2.vhsbcfyheyuzy@sl.local']
2026-07-14 02:12:07,460 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,460 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2 - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,461 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgmzf2.vhsbcfyheyuzy@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,465 - SL - WARNING - 18626 - "/work/email_handler.py:2066" - handle() - 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2 - No such email log
2026-07-14 02:12:07,465 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgmzf2.vhsbcfyheyuzy@sl.local'], takes 0.005751609802246094 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD SIGNED INVALID non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgmzf2.vhsbcfyheyuzy@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:12:07,466 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - 2f48a678-a6bb-41a9-85e7-3ee70ed0d9a2 - set message_id fd565ed2-7edc-4925-a7e0-20b3c81f78bd
2026-07-14 02:12:07,466 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - fd565ed2-7edc-4925-a7e0-20b3c81f78bd - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,466 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - fd565ed2-7edc-4925-a7e0-20b3c81f78bd - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibtgq2cyibsgm4dgmztgjoq.o7mcwnjvxpmua@sl.local']
2026-07-14 02:12:07,466 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - fd565ed2-7edc-4925-a7e0-20b3c81f78bd - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,467 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - fd565ed2-7edc-4925-a7e0-20b3c81f78bd - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,467 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - fd565ed2-7edc-4925-a7e0-20b3c81f78bd - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibtgq2cyibsgm4dgmztgjoq.o7mcwnjvxpmua@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,471 - SL - WARNING - 18626 - "/work/email_handler.py:2309" - handle_DATA() - fd565ed2-7edc-4925-a7e0-20b3c81f78bd - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibtgq2cyibsgm4dgmztgjoq.o7mcwnjvxpmua@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL FWD SIGNED VALID   non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmycyibtgq2cyibsgm4dgmztgjoq.o7mcwnjvxpmua@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:12:07,472 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - fd565ed2-7edc-4925-a7e0-20b3c81f78bd - set message_id 4917b198-f5c4-4e16-9695-83b68de1354d
2026-07-14 02:12:07,472 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - 4917b198-f5c4-4e16-9695-83b68de1354d - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,472 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - 4917b198-f5c4-4e16-9695-83b68de1354d - New message, mail from <>, rctp tos ['bounce_reply+99999999999999+@sl.local']
2026-07-14 02:12:07,472 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - 4917b198-f5c4-4e16-9695-83b68de1354d - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,472 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - 4917b198-f5c4-4e16-9695-83b68de1354d - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,473 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - 4917b198-f5c4-4e16-9695-83b68de1354d - ==>> Handle mail_from:<>, rcpt_tos:['bounce_reply+99999999999999+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,476 - SL - WARNING - 18626 - "/work/email_handler.py:2086" - handle() - 4917b198-f5c4-4e16-9695-83b68de1354d - No such email log
2026-07-14 02:12:07,476 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - 4917b198-f5c4-4e16-9695-83b68de1354d - Finish mail_from <>, rcpt_tos ['bounce_reply+99999999999999+@sl.local'], takes 0.004608631134033203 seconds with return code '550 SL E512 No such email log'<<===
CELL REP OLD  INVALID bounce        mail_from='<>'                   rcpt='bounce_reply+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:12:07,477 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - 4917b198-f5c4-4e16-9695-83b68de1354d - set message_id 1559cde8-4f13-4cf8-a386-dd849ee6069f
2026-07-14 02:12:07,477 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,477 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - New message, mail from <>, rctp tos ['sl.lmysyibtgq3cyibsgm4dgmztgjoq.tymzt3pc4pzge@sl.local']
2026-07-14 02:12:07,477 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,477 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,478 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmysyibtgq3cyibsgm4dgmztgjoq.tymzt3pc4pzge@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,484 - SL - DEBUG - 18626 - "/work/email_handler.py:1862" - handle_bounce() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - handle bounce for <EmailLog 346>, phase=reply, contact=<Contact 155 contact@example.com 759>, alias=<Alias 759 erring_colder311@sl.local>
2026-07-14 02:12:07,485 - SL - DEBUG - 18626 - "/work/email_handler.py:1605" - handle_bounce_reply_phase() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - Handle reply bounce <Mailbox 529 user_ie2z24kpww@mailbox.test> -> <Alias 759 erring_colder311@sl.local> -> <Contact 155 contact@example.com 759>.<EmailLog 346>
2026-07-14 02:12:07,485 - SL - WARNING - 18626 - "/work/email_handler.py:1615" - handle_bounce_reply_phase() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - cannot get bounce info, debug at
2026-07-14 02:12:07,491 - SL - DEBUG - 18626 - "/work/email_handler.py:1640" - handle_bounce_reply_phase() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - Create refused email <Refused Email 115 None 2026-07-21T02:12:07.489758+00:00>
2026-07-14 02:12:07,496 - SL - DEBUG - 18626 - "/work/email_handler.py:1651" - handle_bounce_reply_phase() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - Inform user <User 453 Test User user_ie2z24kpww@mailbox.test> about bounced email sent by <Alias 759 erring_colder311@sl.local> to <Contact 155 contact@example.com 759>
2026-07-14 02:12:07,527 - SL - DEBUG - 18626 - "/work/app/email_utils.py:303" - send_email() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - send email to user_ie2z24kpww@mailbox.test, subject 'Email cannot be sent to contact@example.com from your alias erring_colder311@sl.local'
2026-07-14 02:12:07,532 - SL - DEBUG - 18626 - "/work/app/mail_sender.py:131" - send() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - send email with subject 'Email cannot be sent to contact@example.com from your alias erring_colder311@sl.local', from '"noreply@sl.local" <noreply@sl.local>' to 'user_ie2z24kpww@mailbox.test'
2026-07-14 02:12:07,532 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - Finish mail_from <>, rcpt_tos ['sl.lmysyibtgq3cyibsgm4dgmztgjoq.tymzt3pc4pzge@sl.local'], takes 0.05523562431335449 seconds with return code '250 SL E212 Bounce Reply phase handled'<<===
CELL REP SIGNED VALID   bounce      mail_from='<>'                   rcpt='sl.lmysyibtgq3cyibsgm4dgmztgjoq.tymzt3pc4pzge@sl.local'
     handle_DATA() = '250 SL E212 Bounce Reply phase handled'
2026-07-14 02:12:07,533 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - 1559cde8-4f13-4cf8-a386-dd849ee6069f - set message_id 139d5f86-b0e5-4a4b-88e4-af723c11294e
2026-07-14 02:12:07,533 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,533 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - New message, mail from attacker@evil.example, rctp tos ['sl.lmysyibtgq3cyibsgm4dgmztgjoq.tymzt3pc4pzge@sl.local']
2026-07-14 02:12:07,534 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,534 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,535 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtgq3cyibsgm4dgmztgjoq.tymzt3pc4pzge@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,546 - SL - WARNING - 18626 - "/work/email_handler.py:2309" - handle_DATA() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - email handling fail with error:VERPReply cannot handle email sent to reply VERP, <Alias 759 erring_colder311@sl.local> -> <Contact 155 contact@example.com 759> (<EmailLog 346>, <User 453 Test User user_ie2z24kpww@mailbox.test> mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtgq3cyibsgm4dgmztgjoq.tymzt3pc4pzge@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL REP SIGNED VALID   non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmysyibtgq3cyibsgm4dgmztgjoq.tymzt3pc4pzge@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:12:07,812 - SL - INFO - 18626 - "/work/app/events/event_dispatcher.py:62" - send_event() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:12:07,828 - SL - DEBUG - 18626 - "/work/app/models.py:1459" - generate_random_alias_email() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - generate email knobby_google332@sl.local
2026-07-14 02:12:07,838 - SL - INFO - 18626 - "/work/app/events/event_dispatcher.py:62" - send_event() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:12:07,857 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - 139d5f86-b0e5-4a4b-88e4-af723c11294e - set message_id 819f9379-4984-4616-a086-0560f7bfc532
2026-07-14 02:12:07,857 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - 819f9379-4984-4616-a086-0560f7bfc532 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,857 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - 819f9379-4984-4616-a086-0560f7bfc532 - New message, mail from <>, rctp tos ['sl.lmycyibtgq3syibsgm4dgmztgjoq.hqjay2skoa3vm@sl.local']
2026-07-14 02:12:07,858 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - 819f9379-4984-4616-a086-0560f7bfc532 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,858 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - 819f9379-4984-4616-a086-0560f7bfc532 - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,859 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - 819f9379-4984-4616-a086-0560f7bfc532 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibtgq3syibsgm4dgmztgjoq.hqjay2skoa3vm@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,864 - SL - DEBUG - 18626 - "/work/email_handler.py:1862" - handle_bounce() - 819f9379-4984-4616-a086-0560f7bfc532 - handle bounce for <EmailLog 347>, phase=forward, contact=<Contact 156 c2@example.com 761>, alias=<Alias 761 knobby_google332@sl.local>
2026-07-14 02:12:07,866 - SL - DEBUG - 18626 - "/work/email_handler.py:1870" - handle_bounce() - 819f9379-4984-4616-a086-0560f7bfc532 - User <User 454 Test User user_jrez3f680h@mailbox.test> is not active
2026-07-14 02:12:07,866 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - 819f9379-4984-4616-a086-0560f7bfc532 - Finish mail_from <>, rcpt_tos ['sl.lmycyibtgq3syibsgm4dgmztgjoq.hqjay2skoa3vm@sl.local'], takes 0.00883030891418457 seconds with return code '550 SL E510 so such user'<<===
CELL FWD SIGNED VALID bounce INACTIVE u2.id=454 el=347 is_active=False
     handle_DATA() = '550 SL E510 so such user'
2026-07-14 02:12:07,867 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - 819f9379-4984-4616-a086-0560f7bfc532 - set message_id fedb8c73-d510-4c20-b696-e53776a951b9
2026-07-14 02:12:07,867 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - fedb8c73-d510-4c20-b696-e53776a951b9 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,867 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - fedb8c73-d510-4c20-b696-e53776a951b9 - New message, mail from <>, rctp tos ['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local']
2026-07-14 02:12:07,868 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - fedb8c73-d510-4c20-b696-e53776a951b9 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,868 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - fedb8c73-d510-4c20-b696-e53776a951b9 - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,869 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - fedb8c73-d510-4c20-b696-e53776a951b9 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,871 - SL - DEBUG - 18626 - "/work/email_handler.py:1824" - handle_transactional_bounce() - fedb8c73-d510-4c20-b696-e53776a951b9 - handle transactional bounce sent to sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local
2026-07-14 02:12:07,873 - SL - INFO - 18626 - "/work/email_handler.py:1834" - handle_transactional_bounce() - fedb8c73-d510-4c20-b696-e53776a951b9 - No transactional record for <> -> ['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local']
2026-07-14 02:12:07,873 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - fedb8c73-d510-4c20-b696-e53776a951b9 - Finish mail_from <>, rcpt_tos ['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local'], takes 0.005340099334716797 seconds with return code '250 SL E205 bounce handled'<<===
CELL TXN bounce                     handle_DATA() = '250 SL E205 bounce handled'
2026-07-14 02:12:07,873 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - fedb8c73-d510-4c20-b696-e53776a951b9 - set message_id d4bfce26-7f11-431d-acd3-6cc1caabbb8f
2026-07-14 02:12:07,873 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,874 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - New message, mail from user@corp.example, rctp tos ['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local']
2026-07-14 02:12:07,874 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,874 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,875 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - ==>> Handle mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local'], header_from:user@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,877 - SL - DEBUG - 18626 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:12:07,877 - SL - DEBUG - 18626 - "/work/email_handler.py:2049" - handle() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - Ignore out-of-office for transactional emails. Headers: <bound method Message.items of <email.message.Message object at 0x7a5b287afdc0>>
2026-07-14 02:12:07,878 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - Finish mail_from user@corp.example, rcpt_tos ['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local'], takes 0.004282474517822266 seconds with return code '250 SL E206 Out of office'<<===
CELL TXN out-of-office              handle_DATA() = '250 SL E206 Out of office'
2026-07-14 02:12:07,878 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - d4bfce26-7f11-431d-acd3-6cc1caabbb8f - set message_id 56d440e6-82f6-4c80-a182-ef9cb0455030
2026-07-14 02:12:07,878 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:07,878 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - New message, mail from user@corp.example, rctp tos ['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local']
2026-07-14 02:12:07,879 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:07,879 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:07,880 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - ==>> Handle mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:07,883 - SL - WARNING - 18626 - "/work/email_handler.py:2309" - handle_DATA() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - email handling fail with error:VERPTransactional  mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgms5.nzbnkwpzqf4is@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL TXN non-bounce                 handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:12:08,145 - SL - INFO - 18626 - "/work/app/events/event_dispatcher.py:62" - send_event() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:12:08,156 - SL - DEBUG - 18626 - "/work/app/models.py:1459" - generate_random_alias_email() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - generate email blamer_souses949@sl.local
2026-07-14 02:12:08,165 - SL - INFO - 18626 - "/work/app/events/event_dispatcher.py:62" - send_event() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:12:08,181 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - 56d440e6-82f6-4c80-a182-ef9cb0455030 - set message_id a62e181e-f43f-48dc-8e53-f729391ca3b7
2026-07-14 02:12:08,181 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:08,181 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - New message, mail from bounce+348+@sl.local, rctp tos ['blamer_souses949@sl.local']
2026-07-14 02:12:08,182 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:08,182 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:08,183 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - ==>> Handle mail_from:bounce+348+@sl.local, rcpt_tos:['blamer_souses949@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:08,193 - SL - WARNING - 18626 - "/work/email_handler.py:2110" - handle() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - iCloud bounces <EmailLog 348> <Alias 763 blamer_souses949@sl.local>, saved to
2026-07-14 02:12:08,194 - SL - DEBUG - 18626 - "/work/email_handler.py:1862" - handle_bounce() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - handle bounce for <EmailLog 348>, phase=forward, contact=<Contact 157 c3@example.com 763>, alias=<Alias 763 blamer_souses949@sl.local>
2026-07-14 02:12:08,196 - SL - ERROR - 18626 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - Use <Alias 763 blamer_souses949@sl.local> default mailbox <Mailbox 531 user_j9ma4gwax4@mailbox.test>
NoneType: None
2026-07-14 02:12:08,196 - SL - WARNING - 18626 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - cannot get bounce info, debug at
2026-07-14 02:12:08,197 - SL - DEBUG - 18626 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - Handle forward bounce <Contact 157 c3@example.com 763> -> <Alias 763 blamer_souses949@sl.local> -> <Mailbox 531 user_j9ma4gwax4@mailbox.test>. <EmailLog 348>
2026-07-14 02:12:08,202 - SL - WARNING - 18626 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - Cannot parse original message from bounce message <Alias 763 blamer_souses949@sl.local> <User 455 Test User user_j9ma4gwax4@mailbox.test> <Contact 157 c3@example.com 763> refused-emails/full-ddb30f6c-96e7-4f61-ae35-2b3c5ac93a70.eml
2026-07-14 02:12:08,205 - SL - DEBUG - 18626 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - Create refused email <Refused Email 116 None 2026-07-21T02:12:08.204583+00:00>
2026-07-14 02:12:08,216 - SL - DEBUG - 18626 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - Inform user <User 455 Test User user_j9ma4gwax4@mailbox.test> about a bounce from contact <Contact 157 c3@example.com 763> to alias <Alias 763 blamer_souses949@sl.local>
2026-07-14 02:12:08,248 - SL - DEBUG - 18626 - "/work/app/email_utils.py:303" - send_email() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - send email to user_j9ma4gwax4@mailbox.test, subject 'An email sent to blamer_souses949@sl.local cannot be delivered to your mailbox'
2026-07-14 02:12:08,253 - SL - DEBUG - 18626 - "/work/app/mail_sender.py:131" - send() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - send email with subject 'An email sent to blamer_souses949@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_j9ma4gwax4@mailbox.test'
2026-07-14 02:12:08,254 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - Finish mail_from bounce+348+@sl.local, rcpt_tos ['blamer_souses949@sl.local'], takes 0.0724802017211914 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
CELL iCloud OLD VALID mail_from     mail_from='bounce+348+@sl.local' rcpt='blamer_souses949@sl.local'
     handle_DATA() = '250 SL E211 Bounce Forward phase handled'
2026-07-14 02:12:08,254 - SL - DEBUG - 18626 - "/work/app/log.py:24" - set_message_id() - a62e181e-f43f-48dc-8e53-f729391ca3b7 - set message_id 5cc4ebcf-4fee-47ea-b788-796f709ca915
2026-07-14 02:12:08,254 - SL - DEBUG - 18626 - "/work/email_handler.py:2342" - _handle() - 5cc4ebcf-4fee-47ea-b788-796f709ca915 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:12:08,254 - SL - INFO - 18626 - "/work/email_handler.py:2343" - _handle() - 5cc4ebcf-4fee-47ea-b788-796f709ca915 - New message, mail from bounce+99999999999999+@sl.local, rctp tos ['blamer_souses949@sl.local']
2026-07-14 02:12:08,255 - SL - INFO - 18626 - "/work/email_handler.py:1956" - handle() - 5cc4ebcf-4fee-47ea-b788-796f709ca915 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:12:08,255 - SL - DEBUG - 18626 - "/work/email_handler.py:1963" - handle() - 5cc4ebcf-4fee-47ea-b788-796f709ca915 - Cannot parse Postfix queue ID from None None
2026-07-14 02:12:08,257 - SL - DEBUG - 18626 - "/work/email_handler.py:1980" - handle() - 5cc4ebcf-4fee-47ea-b788-796f709ca915 - ==>> Handle mail_from:bounce+99999999999999+@sl.local, rcpt_tos:['blamer_souses949@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:12:08,269 - SL - WARNING - 18626 - "/work/email_handler.py:2110" - handle() - 5cc4ebcf-4fee-47ea-b788-796f709ca915 - iCloud bounces None <Alias 763 blamer_souses949@sl.local>, saved to
2026-07-14 02:12:08,269 - SL - WARNING - 18626 - "/work/email_handler.py:1857" - handle_bounce() - 5cc4ebcf-4fee-47ea-b788-796f709ca915 - No such email log
2026-07-14 02:12:08,269 - SL - INFO - 18626 - "/work/email_handler.py:2367" - _handle() - 5cc4ebcf-4fee-47ea-b788-796f709ca915 - Finish mail_from bounce+99999999999999+@sl.local, rcpt_tos ['blamer_souses949@sl.local'], takes 0.01500701904296875 seconds with return code '550 SL E512 No such email log'<<===
CELL iCloud OLD INVALID mail_from   mail_from='bounce+99999999999999+@sl.local' rcpt='blamer_souses949@sl.local'
     handle_DATA() = '550 SL E512 No such email log'

===== CANONICAL OUTCOME SUMMARY (handle_DATA wire responses) =====
FWD OLD  INVALID non-bounce        => 550 SL E512 No such email log
FWD OLD  VALID   non-bounce        => 250 SL E213 Unknown email ignored
FWD OLD  VALID   bounce            => 250 SL E211 Bounce Forward phase handled
FWD SIGNED INVALID non-bounce      => 550 SL E512 No such email log
FWD SIGNED VALID   non-bounce      => 250 SL E213 Unknown email ignored
REP OLD  INVALID bounce            => 550 SL E512 No such email log
REP SIGNED VALID   bounce          => 250 SL E212 Bounce Reply phase handled
REP SIGNED VALID   non-bounce      => 250 SL E213 Unknown email ignored
FWD SIGNED VALID bounce INACTIVE   => 550 SL E510 so such user
TXN bounce                         => 250 SL E205 bounce handled
TXN out-of-office                  => 250 SL E206 Out of office
TXN non-bounce                     => 250 SL E213 Unknown email ignored
iCloud OLD VALID mail_from         => 250 SL E211 Bounce Forward phase handled
iCloud OLD INVALID mail_from       => 550 SL E512 No such email log
===== END SUMMARY (14 cells) =====
OBS_MATRIX_DONE
```

**Prior-session per-script ordered-outcome check (Scripts B, C, D, D2, E, F).** Before the canonical oracle matrix was consolidated into `obs_matrix.py`, each earlier script was run twice and the ordered sequence of outcomes (quoted SL status strings + `RAISED` exception classes) was compared run-to-run; the volatile tokens were expected to differ. Those scripts' raw second-run `.out` files were not retained into this session, so the summary below is the recorded outcome-sequence result; the canonical superset of the same behaviour (Scripts B and E) is now proven stable by the embedded multi-run executable comparison above and the full transcripts in K.1a/K.10:

```text
$ # For each earlier canonical script, compare the ordered sequence of OUTCOMES (quoted SL
$ # status strings + RAISED exception classes) between run1 (.out) and run2 (_run2.out).
$ # Volatile tokens (random fixture names, per-run message-ids, minute-bucketed base32
$ # signatures) are expected to differ; only the outcome sequence must match.

Script B: run1 == run2  (33 outcomes identical)  -> STABLE
Script C: run1 == run2  (10 outcomes identical)  -> STABLE
Script D: run1 == run2  (22 outcomes identical)  -> STABLE
Script D2: run1 == run2  (9 outcomes identical)  -> STABLE
Script E: run1 == run2  (14 outcomes identical)  -> STABLE
Script F: run1 == run2  (4 outcomes identical)  -> STABLE
```

### K.1 Canonical environment facts & the three SPF tests

**Environment capture transcript (`obs_ENV.out`):**

```text
=== $ /app/venv/bin/python --version ===
Python 3.10.18

=== $ /app/venv/bin/python -c import aiosmtpd ===
aiosmtpd 1.4.2

=== $ psql (PostgreSQL server version via SELECT version()) ===
PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit

=== $ redis-server --version ===
Redis server v=7.0.15 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=3f20e06e76a2b578

=== $ alembic current / heads (repo alembic) ===
32f25cbf12f6

=== $ public table count ===
77
```

**Command:**

```text
python -m pytest tests/test_email_handler.py -k "prevent_5xx or preserve_5xx" -v --no-header -p no:cacheprovider
```

**Complete verbatim pytest transcript (`obs_SPFTESTS.out`):**

```text
============================= test session starts ==============================
collecting ... collected 23 items / 20 deselected / 3 selected

tests/test_email_handler.py::test_prevent_5xx_from_spf PASSED            [ 33%]
tests/test_email_handler.py::test_preserve_5xx_with_valid_spf PASSED     [ 66%]
tests/test_email_handler.py::test_preserve_5xx_with_no_header PASSED     [100%]

=============================== warnings summary ===============================
../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:121
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:121: DeprecationWarning: pkg_resources is deprecated as an API
    warnings.warn("pkg_resources is deprecated as an API", DeprecationWarning)

../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google.logging')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(parent)

../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('ruamel')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('zope')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

../app/venv/lib/python3.10/site-packages/flask_limiter/errors.py:10
../app/venv/lib/python3.10/site-packages/flask_limiter/errors.py:10
  /app/venv/lib/python3.10/site-packages/flask_limiter/errors.py:10: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    if LooseVersion(werkzeug_version) < LooseVersion("0.9"):  # pragma: no cover

../app/venv/lib/python3.10/site-packages/flask_admin/contrib/__init__.py:2
  /app/venv/lib/python3.10/site-packages/flask_admin/contrib/__init__.py:2: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('flask_admin.contrib')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    __import__('pkg_resources').declare_namespace(__name__)

../app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('flask_admin')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(parent)

../app/venv/lib/python3.10/site-packages/gnupg.py:997
../app/venv/lib/python3.10/site-packages/gnupg.py:997
  /app/venv/lib/python3.10/site-packages/gnupg.py:997: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    rr.setDaemon(True)

../app/venv/lib/python3.10/site-packages/gnupg.py:1004
../app/venv/lib/python3.10/site-packages/gnupg.py:1004
  /app/venv/lib/python3.10/site-packages/gnupg.py:1004: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    dr.setDaemon(True)

../app/venv/lib/python3.10/site-packages/gnupg.py:172
  /app/venv/lib/python3.10/site-packages/gnupg.py:172: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    wr.setDaemon(True)

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
================ 3 passed, 20 deselected, 18 warnings in 0.98s =================
```

### K.1a Canonical PostgreSQL 13 evidence (AAP-mandated major version)

The Agent Action Plan (Section 0.4.1) mandates **PostgreSQL 13** as the datastore. The in-image test database is PostgreSQL 15.13 (Phase A.3), so a genuine **PostgreSQL 13** instance was additionally provisioned and the **complete canonical `handle_DATA` matrix re-run against it, twice**. Every wire response is byte-identical to the PostgreSQL 15 baseline, confirming the oracle is independent of the PostgreSQL major version.

**Provisioning (actual commands and output, `pg13_provision.txt`):**

```text
$ docker ps --filter name=pg13 --format "{{.Image}} {{.Status}}"
postgres:13 Up 9 minutes

$ docker exec sl-app-0 psql -h pg13 -U test -d test -tAc "SELECT version();"
PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit

# Canonical schema replicated from the in-image PG15 test DB (repo alembic upgrade-from-scratch
# is broken repo-wide: it fails identically on fresh PG15 AND PG13 because note_pg_trgm_index
# [migrations/versions/2021_082012_424808e1fe49_.py] runs before gen_email->alias rename
# [2020_031711_e9395fe234a4_.py] under the resolved history, and env.py wraps all migrations
# in a single transaction). Schema + alembic head copied via pg_dump | psql:
$ pg_dump -h localhost -p 15432 -U test --schema-only --no-owner --no-privileges test | psql -h pg13 -U test -d test
$ pg_dump -h localhost -p 15432 -U test --data-only --table=alembic_version test | psql -h pg13 -U test -d test

$ psql -h pg13 -U test -d test -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
$ psql -h pg13 -U test -d test -tAc "SELECT version_num FROM alembic_version;"
32f25cbf12f6
$ psql -h pg13 -U test -d test -tAc "SELECT extname FROM pg_extension ORDER BY extname;"
pg_trgm
plpgsql
uuid-ossp
$ psql -h pg13 -U test -d test -tAc "SELECT indexname FROM pg_indexes WHERE indexname='note_pg_trgm_index';"
note_pg_trgm_index
```

*(Honest disclosure — reported, not normalized:* the repository's `alembic upgrade head` **from an empty database fails identically on a fresh PostgreSQL 15 and a fresh PostgreSQL 13** — `relation "alias" does not exist` while creating `note_pg_trgm_index` — because under the resolved migration history `note_pg_trgm_index` [migrations/versions/2021_082012_424808e1fe49_.py] runs before the `gen_email`->`alias` table rename [migrations/versions/2020_031711_e9395fe234a4_.py], and `migrations/env.py` wraps the whole upgrade in a single transaction that rolls back entirely on failure. This is a pre-existing, repo-wide migration-ordering quirk unrelated to this investigation and not modified here. The canonical **PostgreSQL 15** `test` database in the image is itself already at head `32f25cbf12f6` with all 77 tables, so the PostgreSQL 13 schema was replicated from it via `pg_dump --schema-only | psql` plus the `alembic_version` row — yielding an identical 77-table schema at the same head, `pg_trgm` enabled and `note_pg_trgm_index` present, with zero restore errors.)*

**Invocation form for the PostgreSQL 13 matrix** (identical to every other observation cell except the `DB_URI` override, which `app/config.py`'s `load_dotenv(CONFIG, override=False)` contract honors without touching any file):

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -e DB_URI=postgresql://test:test@pg13:5432/test \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_matrix.py
```

The observation script `obs_matrix.py` is the canonical driver: for every cell it constructs an `aiosmtpd` `Envelope`, sets `mail_from`/`rcpt_tos`/`original_content`, and calls **`email_handler.MailHandler().handle_DATA(None, None, env)`** — never a helper in isolation. Its full source is embedded at the end of this subsection.

**PostgreSQL 13 — database environment facts (run 1, captured through the app's own engine):**

```text
=== DB ENV FACTS ===
DB_URI(host:port/db) = pg13:5432/test
SELECT version()     = PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
alembic head         = 32f25cbf12f6
public table count   = 77
python               = 3.10.18
aiosmtpd             = 1.4.2
EMAIL_DOMAIN         = sl.local
=== END ENV FACTS ===
```

**PostgreSQL 13 — complete verbatim canonical transcript (run 1, `pg13_run1.out`, exit code 0, 211 lines):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/vllkpijvprnzdcmbypje
Upload files to local dir
>>> init logging <<<
2026-07-14 02:15:09,752 - SL - DEBUG - 18908 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-14 02:15:10,865 - SL - INFO - 18908 - "/work/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 02:15:10,868 - SL - INFO - 18908 - "/work/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 02:15:10,870 - SL - INFO - 18908 - "/work/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 02:15:10,871 - SL - DEBUG - 18908 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
=== DB ENV FACTS ===
DB_URI(host:port/db) = pg13:5432/test
SELECT version()     = PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
alembic head         = 32f25cbf12f6
public table count   = 77
python               = 3.10.18
aiosmtpd             = 1.4.2
EMAIL_DOMAIN         = sl.local
=== END ENV FACTS ===

2026-07-14 02:15:11,195 - SL - INFO - 18908 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:11,208 - SL - DEBUG - 18908 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email farmer_smiths304@sl.local
2026-07-14 02:15:11,215 - SL - INFO - 18908 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURES user.id=1 alias=farmer_smiths304@sl.local valid_fwd=(1,2) valid_reply=3 INVALID=99999999999999

2026-07-14 02:15:11,246 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() -  - set message_id 98c7d835-e0ed-4ed1-8fdb-0282cfeba545
2026-07-14 02:15:11,246 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 98c7d835-e0ed-4ed1-8fdb-0282cfeba545 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,246 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 98c7d835-e0ed-4ed1-8fdb-0282cfeba545 - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-14 02:15:11,247 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 98c7d835-e0ed-4ed1-8fdb-0282cfeba545 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,247 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 98c7d835-e0ed-4ed1-8fdb-0282cfeba545 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,248 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 98c7d835-e0ed-4ed1-8fdb-0282cfeba545 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,253 - SL - WARNING - 18908 - "/work/email_handler.py:2066" - handle() - 98c7d835-e0ed-4ed1-8fdb-0282cfeba545 - No such email log
2026-07-14 02:15:11,253 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 98c7d835-e0ed-4ed1-8fdb-0282cfeba545 - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.00670933723449707 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD OLD  INVALID non-bounce    mail_from='attacker@evil.example' rcpt='bounce+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:15:11,253 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 98c7d835-e0ed-4ed1-8fdb-0282cfeba545 - set message_id 51f7e7dd-8d2c-4634-8ce7-961757fcce39
2026-07-14 02:15:11,254 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 51f7e7dd-8d2c-4634-8ce7-961757fcce39 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,254 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 51f7e7dd-8d2c-4634-8ce7-961757fcce39 - New message, mail from attacker@evil.example, rctp tos ['bounce+1+@sl.local']
2026-07-14 02:15:11,254 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 51f7e7dd-8d2c-4634-8ce7-961757fcce39 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,254 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 51f7e7dd-8d2c-4634-8ce7-961757fcce39 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,255 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 51f7e7dd-8d2c-4634-8ce7-961757fcce39 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+1+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,260 - SL - WARNING - 18908 - "/work/email_handler.py:2309" - handle_DATA() - 51f7e7dd-8d2c-4634-8ce7-961757fcce39 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+1+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL FWD OLD  VALID   non-bounce    mail_from='attacker@evil.example' rcpt='bounce+1+@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:15:11,261 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 51f7e7dd-8d2c-4634-8ce7-961757fcce39 - set message_id 908bf80d-0b99-4c34-9ee9-42318ad4170b
2026-07-14 02:15:11,261 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,261 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - New message, mail from <>, rctp tos ['bounce+2+@sl.local']
2026-07-14 02:15:11,262 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,262 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,263 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - ==>> Handle mail_from:<>, rcpt_tos:['bounce+2+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,271 - SL - DEBUG - 18908 - "/work/email_handler.py:1862" - handle_bounce() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - handle bounce for <EmailLog 2>, phase=forward, contact=<Contact 1 contact@example.com 2>, alias=<Alias 2 farmer_smiths304@sl.local>
2026-07-14 02:15:11,273 - SL - ERROR - 18908 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - Use <Alias 2 farmer_smiths304@sl.local> default mailbox <Mailbox 1 user_r4lhhp3i6d@mailbox.test>
NoneType: None
2026-07-14 02:15:11,274 - SL - WARNING - 18908 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - cannot get bounce info, debug at
2026-07-14 02:15:11,276 - SL - DEBUG - 18908 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - Handle forward bounce <Contact 1 contact@example.com 2> -> <Alias 2 farmer_smiths304@sl.local> -> <Mailbox 1 user_r4lhhp3i6d@mailbox.test>. <EmailLog 2>
2026-07-14 02:15:11,281 - SL - WARNING - 18908 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - Cannot parse original message from bounce message <Alias 2 farmer_smiths304@sl.local> <User 1 Test User user_r4lhhp3i6d@mailbox.test> <Contact 1 contact@example.com 2> refused-emails/full-6b18160e-4abe-4a1a-8859-25d1e66498d4.eml
2026-07-14 02:15:11,285 - SL - DEBUG - 18908 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - Create refused email <Refused Email 1 None 2026-07-21T02:15:11.284444+00:00>
2026-07-14 02:15:11,297 - SL - DEBUG - 18908 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - Inform user <User 1 Test User user_r4lhhp3i6d@mailbox.test> about a bounce from contact <Contact 1 contact@example.com 2> to alias <Alias 2 farmer_smiths304@sl.local>
2026-07-14 02:15:11,331 - SL - DEBUG - 18908 - "/work/app/email_utils.py:303" - send_email() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - send email to user_r4lhhp3i6d@mailbox.test, subject 'An email sent to farmer_smiths304@sl.local cannot be delivered to your mailbox'
2026-07-14 02:15:11,337 - SL - DEBUG - 18908 - "/work/app/mail_sender.py:131" - send() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - send email with subject 'An email sent to farmer_smiths304@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_r4lhhp3i6d@mailbox.test'
2026-07-14 02:15:11,337 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - Finish mail_from <>, rcpt_tos ['bounce+2+@sl.local'], takes 0.07598042488098145 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
CELL FWD OLD  VALID   bounce        mail_from='<>'                   rcpt='bounce+2+@sl.local'
     handle_DATA() = '250 SL E211 Bounce Forward phase handled'
2026-07-14 02:15:11,338 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 908bf80d-0b99-4c34-9ee9-42318ad4170b - set message_id 4b7c5fe0-177b-47a8-94ca-5f6dd014822a
2026-07-14 02:15:11,338 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 4b7c5fe0-177b-47a8-94ca-5f6dd014822a - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,338 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 4b7c5fe0-177b-47a8-94ca-5f6dd014822a - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgm2v2.xmdjbwyavor3q@sl.local']
2026-07-14 02:15:11,339 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 4b7c5fe0-177b-47a8-94ca-5f6dd014822a - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,339 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 4b7c5fe0-177b-47a8-94ca-5f6dd014822a - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,340 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 4b7c5fe0-177b-47a8-94ca-5f6dd014822a - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgm2v2.xmdjbwyavor3q@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,344 - SL - WARNING - 18908 - "/work/email_handler.py:2066" - handle() - 4b7c5fe0-177b-47a8-94ca-5f6dd014822a - No such email log
2026-07-14 02:15:11,344 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 4b7c5fe0-177b-47a8-94ca-5f6dd014822a - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgm2v2.xmdjbwyavor3q@sl.local'], takes 0.0061397552490234375 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD SIGNED INVALID non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgm2v2.xmdjbwyavor3q@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:15:11,345 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 4b7c5fe0-177b-47a8-94ca-5f6dd014822a - set message_id 244a7960-97a2-4cec-8224-d1bfd5b6c956
2026-07-14 02:15:11,345 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 244a7960-97a2-4cec-8224-d1bfd5b6c956 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,345 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 244a7960-97a2-4cec-8224-d1bfd5b6c956 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibrfqqdemzygmztgnk5.vzask5huqc2zo@sl.local']
2026-07-14 02:15:11,346 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 244a7960-97a2-4cec-8224-d1bfd5b6c956 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,346 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 244a7960-97a2-4cec-8224-d1bfd5b6c956 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,347 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 244a7960-97a2-4cec-8224-d1bfd5b6c956 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibrfqqdemzygmztgnk5.vzask5huqc2zo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,351 - SL - WARNING - 18908 - "/work/email_handler.py:2309" - handle_DATA() - 244a7960-97a2-4cec-8224-d1bfd5b6c956 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibrfqqdemzygmztgnk5.vzask5huqc2zo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL FWD SIGNED VALID   non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmycyibrfqqdemzygmztgnk5.vzask5huqc2zo@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:15:11,351 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 244a7960-97a2-4cec-8224-d1bfd5b6c956 - set message_id 13b3e399-bf73-4aea-9f9e-190eb99f84fd
2026-07-14 02:15:11,351 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 13b3e399-bf73-4aea-9f9e-190eb99f84fd - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,351 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 13b3e399-bf73-4aea-9f9e-190eb99f84fd - New message, mail from <>, rctp tos ['bounce_reply+99999999999999+@sl.local']
2026-07-14 02:15:11,352 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 13b3e399-bf73-4aea-9f9e-190eb99f84fd - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,352 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 13b3e399-bf73-4aea-9f9e-190eb99f84fd - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,353 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 13b3e399-bf73-4aea-9f9e-190eb99f84fd - ==>> Handle mail_from:<>, rcpt_tos:['bounce_reply+99999999999999+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,357 - SL - WARNING - 18908 - "/work/email_handler.py:2086" - handle() - 13b3e399-bf73-4aea-9f9e-190eb99f84fd - No such email log
2026-07-14 02:15:11,357 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 13b3e399-bf73-4aea-9f9e-190eb99f84fd - Finish mail_from <>, rcpt_tos ['bounce_reply+99999999999999+@sl.local'], takes 0.005424976348876953 seconds with return code '550 SL E512 No such email log'<<===
CELL REP OLD  INVALID bounce        mail_from='<>'                   rcpt='bounce_reply+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:15:11,357 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 13b3e399-bf73-4aea-9f9e-190eb99f84fd - set message_id 5221e8b1-78e1-4a46-8203-69cd0c1c708d
2026-07-14 02:15:11,358 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,358 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - New message, mail from <>, rctp tos ['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local']
2026-07-14 02:15:11,358 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,358 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,359 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,365 - SL - DEBUG - 18908 - "/work/email_handler.py:1862" - handle_bounce() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - handle bounce for <EmailLog 3>, phase=reply, contact=<Contact 1 contact@example.com 2>, alias=<Alias 2 farmer_smiths304@sl.local>
2026-07-14 02:15:11,366 - SL - DEBUG - 18908 - "/work/email_handler.py:1605" - handle_bounce_reply_phase() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - Handle reply bounce <Mailbox 1 user_r4lhhp3i6d@mailbox.test> -> <Alias 2 farmer_smiths304@sl.local> -> <Contact 1 contact@example.com 2>.<EmailLog 3>
2026-07-14 02:15:11,366 - SL - WARNING - 18908 - "/work/email_handler.py:1615" - handle_bounce_reply_phase() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - cannot get bounce info, debug at
2026-07-14 02:15:11,372 - SL - DEBUG - 18908 - "/work/email_handler.py:1640" - handle_bounce_reply_phase() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - Create refused email <Refused Email 2 None 2026-07-21T02:15:11.370640+00:00>
2026-07-14 02:15:11,377 - SL - DEBUG - 18908 - "/work/email_handler.py:1651" - handle_bounce_reply_phase() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - Inform user <User 1 Test User user_r4lhhp3i6d@mailbox.test> about bounced email sent by <Alias 2 farmer_smiths304@sl.local> to <Contact 1 contact@example.com 2>
2026-07-14 02:15:11,402 - SL - DEBUG - 18908 - "/work/app/email_utils.py:303" - send_email() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - send email to user_r4lhhp3i6d@mailbox.test, subject 'Email cannot be sent to contact@example.com from your alias farmer_smiths304@sl.local'
2026-07-14 02:15:11,406 - SL - DEBUG - 18908 - "/work/app/mail_sender.py:131" - send() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - send email with subject 'Email cannot be sent to contact@example.com from your alias farmer_smiths304@sl.local', from '"noreply@sl.local" <noreply@sl.local>' to 'user_r4lhhp3i6d@mailbox.test'
2026-07-14 02:15:11,406 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - Finish mail_from <>, rcpt_tos ['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'], takes 0.04861044883728027 seconds with return code '250 SL E212 Bounce Reply phase handled'<<===
CELL REP SIGNED VALID   bounce      mail_from='<>'                   rcpt='sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'
     handle_DATA() = '250 SL E212 Bounce Reply phase handled'
2026-07-14 02:15:11,407 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 5221e8b1-78e1-4a46-8203-69cd0c1c708d - set message_id e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba
2026-07-14 02:15:11,407 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,407 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - New message, mail from attacker@evil.example, rctp tos ['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local']
2026-07-14 02:15:11,408 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,408 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,409 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,418 - SL - WARNING - 18908 - "/work/email_handler.py:2309" - handle_DATA() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - email handling fail with error:VERPReply cannot handle email sent to reply VERP, <Alias 2 farmer_smiths304@sl.local> -> <Contact 1 contact@example.com 2> (<EmailLog 3>, <User 1 Test User user_r4lhhp3i6d@mailbox.test> mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL REP SIGNED VALID   non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:15:11,681 - SL - INFO - 18908 - "/work/app/events/event_dispatcher.py:62" - send_event() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:11,692 - SL - DEBUG - 18908 - "/work/app/models.py:1459" - generate_random_alias_email() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - generate email horsey_blazer183@sl.local
2026-07-14 02:15:11,699 - SL - INFO - 18908 - "/work/app/events/event_dispatcher.py:62" - send_event() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:11,716 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - e7a15d7a-4e6c-4d48-98ab-a3af228ec8ba - set message_id 032518b2-219a-4be4-b4d7-321edcdbd0b1
2026-07-14 02:15:11,716 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,716 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - New message, mail from <>, rctp tos ['sl.lmycyibufqqdemzygmztgnk5.s54xkgg3su6yq@sl.local']
2026-07-14 02:15:11,717 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,717 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,717 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibufqqdemzygmztgnk5.s54xkgg3su6yq@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,721 - SL - DEBUG - 18908 - "/work/email_handler.py:1862" - handle_bounce() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - handle bounce for <EmailLog 4>, phase=forward, contact=<Contact 2 c2@example.com 4>, alias=<Alias 4 horsey_blazer183@sl.local>
2026-07-14 02:15:11,722 - SL - DEBUG - 18908 - "/work/email_handler.py:1870" - handle_bounce() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - User <User 2 Test User user_w7g2jg732y@mailbox.test> is not active
2026-07-14 02:15:11,722 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - Finish mail_from <>, rcpt_tos ['sl.lmycyibufqqdemzygmztgnk5.s54xkgg3su6yq@sl.local'], takes 0.006525754928588867 seconds with return code '550 SL E510 so such user'<<===
CELL FWD SIGNED VALID bounce INACTIVE u2.id=2 el=4 is_active=False
     handle_DATA() = '550 SL E510 so such user'
2026-07-14 02:15:11,723 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 032518b2-219a-4be4-b4d7-321edcdbd0b1 - set message_id 57e4456f-3dc5-4ca2-9a7e-c550732b2b72
2026-07-14 02:15:11,723 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,723 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - New message, mail from <>, rctp tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local']
2026-07-14 02:15:11,724 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,724 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,725 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,726 - SL - DEBUG - 18908 - "/work/email_handler.py:1824" - handle_transactional_bounce() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - handle transactional bounce sent to sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local
2026-07-14 02:15:11,727 - SL - INFO - 18908 - "/work/email_handler.py:1834" - handle_transactional_bounce() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - No transactional record for <> -> ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local']
2026-07-14 02:15:11,727 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - Finish mail_from <>, rcpt_tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], takes 0.004292011260986328 seconds with return code '250 SL E205 bounce handled'<<===
CELL TXN bounce                     handle_DATA() = '250 SL E205 bounce handled'
2026-07-14 02:15:11,728 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 57e4456f-3dc5-4ca2-9a7e-c550732b2b72 - set message_id 47ff4c8e-eb9d-488b-a7b1-649a188e631a
2026-07-14 02:15:11,728 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,728 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - New message, mail from user@corp.example, rctp tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local']
2026-07-14 02:15:11,729 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,729 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,730 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - ==>> Handle mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], header_from:user@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,731 - SL - DEBUG - 18908 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:15:11,731 - SL - DEBUG - 18908 - "/work/email_handler.py:2049" - handle() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - Ignore out-of-office for transactional emails. Headers: <bound method Message.items of <email.message.Message object at 0x78436570d360>>
2026-07-14 02:15:11,732 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - Finish mail_from user@corp.example, rcpt_tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], takes 0.0035173892974853516 seconds with return code '250 SL E206 Out of office'<<===
CELL TXN out-of-office              handle_DATA() = '250 SL E206 Out of office'
2026-07-14 02:15:11,732 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 47ff4c8e-eb9d-488b-a7b1-649a188e631a - set message_id 99288a1a-8051-4f21-bb31-c670cb2047b2
2026-07-14 02:15:11,732 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:11,732 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - New message, mail from user@corp.example, rctp tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local']
2026-07-14 02:15:11,733 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:11,733 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:11,734 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - ==>> Handle mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:11,736 - SL - WARNING - 18908 - "/work/email_handler.py:2309" - handle_DATA() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - email handling fail with error:VERPTransactional  mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL TXN non-bounce                 handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:15:11,999 - SL - INFO - 18908 - "/work/app/events/event_dispatcher.py:62" - send_event() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:12,012 - SL - DEBUG - 18908 - "/work/app/models.py:1459" - generate_random_alias_email() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - generate email besots_panama556@sl.local
2026-07-14 02:15:12,021 - SL - INFO - 18908 - "/work/app/events/event_dispatcher.py:62" - send_event() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:12,037 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 99288a1a-8051-4f21-bb31-c670cb2047b2 - set message_id 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26
2026-07-14 02:15:12,037 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:12,037 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - New message, mail from bounce+5+@sl.local, rctp tos ['besots_panama556@sl.local']
2026-07-14 02:15:12,038 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:12,038 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:12,039 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - ==>> Handle mail_from:bounce+5+@sl.local, rcpt_tos:['besots_panama556@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:12,048 - SL - WARNING - 18908 - "/work/email_handler.py:2110" - handle() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - iCloud bounces <EmailLog 5> <Alias 6 besots_panama556@sl.local>, saved to
2026-07-14 02:15:12,049 - SL - DEBUG - 18908 - "/work/email_handler.py:1862" - handle_bounce() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - handle bounce for <EmailLog 5>, phase=forward, contact=<Contact 3 c3@example.com 6>, alias=<Alias 6 besots_panama556@sl.local>
2026-07-14 02:15:12,051 - SL - ERROR - 18908 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - Use <Alias 6 besots_panama556@sl.local> default mailbox <Mailbox 3 user_1lgj5jsp5z@mailbox.test>
NoneType: None
2026-07-14 02:15:12,051 - SL - WARNING - 18908 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - cannot get bounce info, debug at
2026-07-14 02:15:12,053 - SL - DEBUG - 18908 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - Handle forward bounce <Contact 3 c3@example.com 6> -> <Alias 6 besots_panama556@sl.local> -> <Mailbox 3 user_1lgj5jsp5z@mailbox.test>. <EmailLog 5>
2026-07-14 02:15:12,056 - SL - WARNING - 18908 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - Cannot parse original message from bounce message <Alias 6 besots_panama556@sl.local> <User 3 Test User user_1lgj5jsp5z@mailbox.test> <Contact 3 c3@example.com 6> refused-emails/full-e23afe53-a79e-4fd3-b540-1c591bbd95cd.eml
2026-07-14 02:15:12,059 - SL - DEBUG - 18908 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - Create refused email <Refused Email 3 None 2026-07-21T02:15:12.058712+00:00>
2026-07-14 02:15:12,069 - SL - DEBUG - 18908 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - Inform user <User 3 Test User user_1lgj5jsp5z@mailbox.test> about a bounce from contact <Contact 3 c3@example.com 6> to alias <Alias 6 besots_panama556@sl.local>
2026-07-14 02:15:12,100 - SL - DEBUG - 18908 - "/work/app/email_utils.py:303" - send_email() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - send email to user_1lgj5jsp5z@mailbox.test, subject 'An email sent to besots_panama556@sl.local cannot be delivered to your mailbox'
2026-07-14 02:15:12,106 - SL - DEBUG - 18908 - "/work/app/mail_sender.py:131" - send() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - send email with subject 'An email sent to besots_panama556@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_1lgj5jsp5z@mailbox.test'
2026-07-14 02:15:12,106 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - Finish mail_from bounce+5+@sl.local, rcpt_tos ['besots_panama556@sl.local'], takes 0.06980347633361816 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
CELL iCloud OLD VALID mail_from     mail_from='bounce+5+@sl.local' rcpt='besots_panama556@sl.local'
     handle_DATA() = '250 SL E211 Bounce Forward phase handled'
2026-07-14 02:15:12,107 - SL - DEBUG - 18908 - "/work/app/log.py:24" - set_message_id() - 3f594c1d-0888-4cf2-9a9c-d4d4c0678c26 - set message_id 304a7d32-1c55-4b67-83ed-96cff42b8998
2026-07-14 02:15:12,107 - SL - DEBUG - 18908 - "/work/email_handler.py:2342" - _handle() - 304a7d32-1c55-4b67-83ed-96cff42b8998 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:12,107 - SL - INFO - 18908 - "/work/email_handler.py:2343" - _handle() - 304a7d32-1c55-4b67-83ed-96cff42b8998 - New message, mail from bounce+99999999999999+@sl.local, rctp tos ['besots_panama556@sl.local']
2026-07-14 02:15:12,108 - SL - INFO - 18908 - "/work/email_handler.py:1956" - handle() - 304a7d32-1c55-4b67-83ed-96cff42b8998 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:12,108 - SL - DEBUG - 18908 - "/work/email_handler.py:1963" - handle() - 304a7d32-1c55-4b67-83ed-96cff42b8998 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:12,110 - SL - DEBUG - 18908 - "/work/email_handler.py:1980" - handle() - 304a7d32-1c55-4b67-83ed-96cff42b8998 - ==>> Handle mail_from:bounce+99999999999999+@sl.local, rcpt_tos:['besots_panama556@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:12,122 - SL - WARNING - 18908 - "/work/email_handler.py:2110" - handle() - 304a7d32-1c55-4b67-83ed-96cff42b8998 - iCloud bounces None <Alias 6 besots_panama556@sl.local>, saved to
2026-07-14 02:15:12,122 - SL - WARNING - 18908 - "/work/email_handler.py:1857" - handle_bounce() - 304a7d32-1c55-4b67-83ed-96cff42b8998 - No such email log
2026-07-14 02:15:12,122 - SL - INFO - 18908 - "/work/email_handler.py:2367" - _handle() - 304a7d32-1c55-4b67-83ed-96cff42b8998 - Finish mail_from bounce+99999999999999+@sl.local, rcpt_tos ['besots_panama556@sl.local'], takes 0.01449441909790039 seconds with return code '550 SL E512 No such email log'<<===
CELL iCloud OLD INVALID mail_from   mail_from='bounce+99999999999999+@sl.local' rcpt='besots_panama556@sl.local'
     handle_DATA() = '550 SL E512 No such email log'

===== CANONICAL OUTCOME SUMMARY (handle_DATA wire responses) =====
FWD OLD  INVALID non-bounce        => 550 SL E512 No such email log
FWD OLD  VALID   non-bounce        => 250 SL E213 Unknown email ignored
FWD OLD  VALID   bounce            => 250 SL E211 Bounce Forward phase handled
FWD SIGNED INVALID non-bounce      => 550 SL E512 No such email log
FWD SIGNED VALID   non-bounce      => 250 SL E213 Unknown email ignored
REP OLD  INVALID bounce            => 550 SL E512 No such email log
REP SIGNED VALID   bounce          => 250 SL E212 Bounce Reply phase handled
REP SIGNED VALID   non-bounce      => 250 SL E213 Unknown email ignored
FWD SIGNED VALID bounce INACTIVE   => 550 SL E510 so such user
TXN bounce                         => 250 SL E205 bounce handled
TXN out-of-office                  => 250 SL E206 Out of office
TXN non-bounce                     => 250 SL E213 Unknown email ignored
iCloud OLD VALID mail_from         => 250 SL E211 Bounce Forward phase handled
iCloud OLD INVALID mail_from       => 550 SL E512 No such email log
===== END SUMMARY (14 cells) =====
OBS_MATRIX_DONE
```

**PostgreSQL 13 — complete verbatim canonical transcript (run 2, `pg13_run2.out`, exit code 0, 211 lines; database reset to pristine schema before this run — `SELECT count(*) FROM email_log = 0`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/sugfyjvajgltduqxvyoy
Upload files to local dir
>>> init logging <<<
2026-07-14 02:15:27,693 - SL - DEBUG - 18969 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-14 02:15:28,774 - SL - INFO - 18969 - "/work/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 02:15:28,777 - SL - INFO - 18969 - "/work/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 02:15:28,778 - SL - INFO - 18969 - "/work/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 02:15:28,779 - SL - DEBUG - 18969 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
=== DB ENV FACTS ===
DB_URI(host:port/db) = pg13:5432/test
SELECT version()     = PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
alembic head         = 32f25cbf12f6
public table count   = 77
python               = 3.10.18
aiosmtpd             = 1.4.2
EMAIL_DOMAIN         = sl.local
=== END ENV FACTS ===

2026-07-14 02:15:29,080 - SL - INFO - 18969 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:29,094 - SL - DEBUG - 18969 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email fudges_refuel774@sl.local
2026-07-14 02:15:29,102 - SL - INFO - 18969 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURES user.id=1 alias=fudges_refuel774@sl.local valid_fwd=(1,2) valid_reply=3 INVALID=99999999999999

2026-07-14 02:15:29,135 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() -  - set message_id 1693249f-2321-423a-84e7-74c2fbdd31f7
2026-07-14 02:15:29,135 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - 1693249f-2321-423a-84e7-74c2fbdd31f7 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,135 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - 1693249f-2321-423a-84e7-74c2fbdd31f7 - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-14 02:15:29,136 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - 1693249f-2321-423a-84e7-74c2fbdd31f7 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,136 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - 1693249f-2321-423a-84e7-74c2fbdd31f7 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,138 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - 1693249f-2321-423a-84e7-74c2fbdd31f7 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,142 - SL - WARNING - 18969 - "/work/email_handler.py:2066" - handle() - 1693249f-2321-423a-84e7-74c2fbdd31f7 - No such email log
2026-07-14 02:15:29,142 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - 1693249f-2321-423a-84e7-74c2fbdd31f7 - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.007106304168701172 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD OLD  INVALID non-bounce    mail_from='attacker@evil.example' rcpt='bounce+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:15:29,143 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - 1693249f-2321-423a-84e7-74c2fbdd31f7 - set message_id 1ad9e8c6-485d-43aa-bab2-6d48e04de1ed
2026-07-14 02:15:29,143 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - 1ad9e8c6-485d-43aa-bab2-6d48e04de1ed - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,143 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - 1ad9e8c6-485d-43aa-bab2-6d48e04de1ed - New message, mail from attacker@evil.example, rctp tos ['bounce+1+@sl.local']
2026-07-14 02:15:29,144 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - 1ad9e8c6-485d-43aa-bab2-6d48e04de1ed - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,144 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - 1ad9e8c6-485d-43aa-bab2-6d48e04de1ed - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,145 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - 1ad9e8c6-485d-43aa-bab2-6d48e04de1ed - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+1+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,149 - SL - WARNING - 18969 - "/work/email_handler.py:2309" - handle_DATA() - 1ad9e8c6-485d-43aa-bab2-6d48e04de1ed - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+1+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL FWD OLD  VALID   non-bounce    mail_from='attacker@evil.example' rcpt='bounce+1+@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:15:29,150 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - 1ad9e8c6-485d-43aa-bab2-6d48e04de1ed - set message_id a9c3f23d-092d-4497-8ad1-aa6474777d6e
2026-07-14 02:15:29,150 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,150 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - New message, mail from <>, rctp tos ['bounce+2+@sl.local']
2026-07-14 02:15:29,150 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,150 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,151 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - ==>> Handle mail_from:<>, rcpt_tos:['bounce+2+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,159 - SL - DEBUG - 18969 - "/work/email_handler.py:1862" - handle_bounce() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - handle bounce for <EmailLog 2>, phase=forward, contact=<Contact 1 contact@example.com 2>, alias=<Alias 2 fudges_refuel774@sl.local>
2026-07-14 02:15:29,160 - SL - ERROR - 18969 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - Use <Alias 2 fudges_refuel774@sl.local> default mailbox <Mailbox 1 user_7ibq1qm7kq@mailbox.test>
NoneType: None
2026-07-14 02:15:29,161 - SL - WARNING - 18969 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - cannot get bounce info, debug at
2026-07-14 02:15:29,209 - SL - DEBUG - 18969 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - Handle forward bounce <Contact 1 contact@example.com 2> -> <Alias 2 fudges_refuel774@sl.local> -> <Mailbox 1 user_7ibq1qm7kq@mailbox.test>. <EmailLog 2>
2026-07-14 02:15:29,214 - SL - WARNING - 18969 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - Cannot parse original message from bounce message <Alias 2 fudges_refuel774@sl.local> <User 1 Test User user_7ibq1qm7kq@mailbox.test> <Contact 1 contact@example.com 2> refused-emails/full-a869dc55-ff35-4761-8dab-cf1d5c4ae8d5.eml
2026-07-14 02:15:29,218 - SL - DEBUG - 18969 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - Create refused email <Refused Email 1 None 2026-07-21T02:15:29.217579+00:00>
2026-07-14 02:15:29,229 - SL - DEBUG - 18969 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - Inform user <User 1 Test User user_7ibq1qm7kq@mailbox.test> about a bounce from contact <Contact 1 contact@example.com 2> to alias <Alias 2 fudges_refuel774@sl.local>
2026-07-14 02:15:29,259 - SL - DEBUG - 18969 - "/work/app/email_utils.py:303" - send_email() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - send email to user_7ibq1qm7kq@mailbox.test, subject 'An email sent to fudges_refuel774@sl.local cannot be delivered to your mailbox'
2026-07-14 02:15:29,263 - SL - DEBUG - 18969 - "/work/app/mail_sender.py:131" - send() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - send email with subject 'An email sent to fudges_refuel774@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_7ibq1qm7kq@mailbox.test'
2026-07-14 02:15:29,263 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - Finish mail_from <>, rcpt_tos ['bounce+2+@sl.local'], takes 0.11370396614074707 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
CELL FWD OLD  VALID   bounce        mail_from='<>'                   rcpt='bounce+2+@sl.local'
     handle_DATA() = '250 SL E211 Bounce Forward phase handled'
2026-07-14 02:15:29,264 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - a9c3f23d-092d-4497-8ad1-aa6474777d6e - set message_id 117226b9-ede4-4b78-89c6-4655f3dfb3b1
2026-07-14 02:15:29,264 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - 117226b9-ede4-4b78-89c6-4655f3dfb3b1 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,264 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - 117226b9-ede4-4b78-89c6-4655f3dfb3b1 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgm2v2.xmdjbwyavor3q@sl.local']
2026-07-14 02:15:29,265 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - 117226b9-ede4-4b78-89c6-4655f3dfb3b1 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,265 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - 117226b9-ede4-4b78-89c6-4655f3dfb3b1 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,266 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - 117226b9-ede4-4b78-89c6-4655f3dfb3b1 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgm2v2.xmdjbwyavor3q@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,269 - SL - WARNING - 18969 - "/work/email_handler.py:2066" - handle() - 117226b9-ede4-4b78-89c6-4655f3dfb3b1 - No such email log
2026-07-14 02:15:29,269 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - 117226b9-ede4-4b78-89c6-4655f3dfb3b1 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgm2v2.xmdjbwyavor3q@sl.local'], takes 0.0051708221435546875 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD SIGNED INVALID non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgm2v2.xmdjbwyavor3q@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:15:29,270 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - 117226b9-ede4-4b78-89c6-4655f3dfb3b1 - set message_id cdf8752e-4c0d-4011-b375-deb5d0f2d764
2026-07-14 02:15:29,270 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - cdf8752e-4c0d-4011-b375-deb5d0f2d764 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,270 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - cdf8752e-4c0d-4011-b375-deb5d0f2d764 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibrfqqdemzygmztgnk5.vzask5huqc2zo@sl.local']
2026-07-14 02:15:29,271 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - cdf8752e-4c0d-4011-b375-deb5d0f2d764 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,271 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - cdf8752e-4c0d-4011-b375-deb5d0f2d764 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,272 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - cdf8752e-4c0d-4011-b375-deb5d0f2d764 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibrfqqdemzygmztgnk5.vzask5huqc2zo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,275 - SL - WARNING - 18969 - "/work/email_handler.py:2309" - handle_DATA() - cdf8752e-4c0d-4011-b375-deb5d0f2d764 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibrfqqdemzygmztgnk5.vzask5huqc2zo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL FWD SIGNED VALID   non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmycyibrfqqdemzygmztgnk5.vzask5huqc2zo@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:15:29,275 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - cdf8752e-4c0d-4011-b375-deb5d0f2d764 - set message_id 0b11d6b4-8f15-497b-be7e-20cbeec83b0a
2026-07-14 02:15:29,275 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - 0b11d6b4-8f15-497b-be7e-20cbeec83b0a - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,275 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - 0b11d6b4-8f15-497b-be7e-20cbeec83b0a - New message, mail from <>, rctp tos ['bounce_reply+99999999999999+@sl.local']
2026-07-14 02:15:29,276 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - 0b11d6b4-8f15-497b-be7e-20cbeec83b0a - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,276 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - 0b11d6b4-8f15-497b-be7e-20cbeec83b0a - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,277 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - 0b11d6b4-8f15-497b-be7e-20cbeec83b0a - ==>> Handle mail_from:<>, rcpt_tos:['bounce_reply+99999999999999+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,280 - SL - WARNING - 18969 - "/work/email_handler.py:2086" - handle() - 0b11d6b4-8f15-497b-be7e-20cbeec83b0a - No such email log
2026-07-14 02:15:29,280 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - 0b11d6b4-8f15-497b-be7e-20cbeec83b0a - Finish mail_from <>, rcpt_tos ['bounce_reply+99999999999999+@sl.local'], takes 0.004477977752685547 seconds with return code '550 SL E512 No such email log'<<===
CELL REP OLD  INVALID bounce        mail_from='<>'                   rcpt='bounce_reply+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:15:29,281 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - 0b11d6b4-8f15-497b-be7e-20cbeec83b0a - set message_id be12151b-da0b-480d-af65-db2955c49d61
2026-07-14 02:15:29,281 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - be12151b-da0b-480d-af65-db2955c49d61 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,281 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - be12151b-da0b-480d-af65-db2955c49d61 - New message, mail from <>, rctp tos ['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local']
2026-07-14 02:15:29,281 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - be12151b-da0b-480d-af65-db2955c49d61 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,281 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - be12151b-da0b-480d-af65-db2955c49d61 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,282 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - be12151b-da0b-480d-af65-db2955c49d61 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,287 - SL - DEBUG - 18969 - "/work/email_handler.py:1862" - handle_bounce() - be12151b-da0b-480d-af65-db2955c49d61 - handle bounce for <EmailLog 3>, phase=reply, contact=<Contact 1 contact@example.com 2>, alias=<Alias 2 fudges_refuel774@sl.local>
2026-07-14 02:15:29,287 - SL - DEBUG - 18969 - "/work/email_handler.py:1605" - handle_bounce_reply_phase() - be12151b-da0b-480d-af65-db2955c49d61 - Handle reply bounce <Mailbox 1 user_7ibq1qm7kq@mailbox.test> -> <Alias 2 fudges_refuel774@sl.local> -> <Contact 1 contact@example.com 2>.<EmailLog 3>
2026-07-14 02:15:29,287 - SL - WARNING - 18969 - "/work/email_handler.py:1615" - handle_bounce_reply_phase() - be12151b-da0b-480d-af65-db2955c49d61 - cannot get bounce info, debug at
2026-07-14 02:15:29,293 - SL - DEBUG - 18969 - "/work/email_handler.py:1640" - handle_bounce_reply_phase() - be12151b-da0b-480d-af65-db2955c49d61 - Create refused email <Refused Email 2 None 2026-07-21T02:15:29.291724+00:00>
2026-07-14 02:15:29,297 - SL - DEBUG - 18969 - "/work/email_handler.py:1651" - handle_bounce_reply_phase() - be12151b-da0b-480d-af65-db2955c49d61 - Inform user <User 1 Test User user_7ibq1qm7kq@mailbox.test> about bounced email sent by <Alias 2 fudges_refuel774@sl.local> to <Contact 1 contact@example.com 2>
2026-07-14 02:15:29,322 - SL - DEBUG - 18969 - "/work/app/email_utils.py:303" - send_email() - be12151b-da0b-480d-af65-db2955c49d61 - send email to user_7ibq1qm7kq@mailbox.test, subject 'Email cannot be sent to contact@example.com from your alias fudges_refuel774@sl.local'
2026-07-14 02:15:29,327 - SL - DEBUG - 18969 - "/work/app/mail_sender.py:131" - send() - be12151b-da0b-480d-af65-db2955c49d61 - send email with subject 'Email cannot be sent to contact@example.com from your alias fudges_refuel774@sl.local', from '"noreply@sl.local" <noreply@sl.local>' to 'user_7ibq1qm7kq@mailbox.test'
2026-07-14 02:15:29,327 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - be12151b-da0b-480d-af65-db2955c49d61 - Finish mail_from <>, rcpt_tos ['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'], takes 0.04674506187438965 seconds with return code '250 SL E212 Bounce Reply phase handled'<<===
CELL REP SIGNED VALID   bounce      mail_from='<>'                   rcpt='sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'
     handle_DATA() = '250 SL E212 Bounce Reply phase handled'
2026-07-14 02:15:29,329 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - be12151b-da0b-480d-af65-db2955c49d61 - set message_id d6101b20-1a5d-426d-b4d4-bd7db5da20dc
2026-07-14 02:15:29,329 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,329 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - New message, mail from attacker@evil.example, rctp tos ['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local']
2026-07-14 02:15:29,329 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,329 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,331 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,339 - SL - WARNING - 18969 - "/work/email_handler.py:2309" - handle_DATA() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - email handling fail with error:VERPReply cannot handle email sent to reply VERP, <Alias 2 fudges_refuel774@sl.local> -> <Contact 1 contact@example.com 2> (<EmailLog 3>, <User 1 Test User user_7ibq1qm7kq@mailbox.test> mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL REP SIGNED VALID   non-bounce  mail_from='attacker@evil.example' rcpt='sl.lmysyibtfqqdemzygmztgnk5.ffzh343lqvyoo@sl.local'
     handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:15:29,602 - SL - INFO - 18969 - "/work/app/events/event_dispatcher.py:62" - send_event() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:29,614 - SL - DEBUG - 18969 - "/work/app/models.py:1459" - generate_random_alias_email() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - generate email papist_runner280@sl.local
2026-07-14 02:15:29,622 - SL - INFO - 18969 - "/work/app/events/event_dispatcher.py:62" - send_event() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:29,639 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - d6101b20-1a5d-426d-b4d4-bd7db5da20dc - set message_id c0c5937a-51a7-4891-afdf-2fd59a99d6d7
2026-07-14 02:15:29,639 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,639 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - New message, mail from <>, rctp tos ['sl.lmycyibufqqdemzygmztgnk5.s54xkgg3su6yq@sl.local']
2026-07-14 02:15:29,640 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,640 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,641 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibufqqdemzygmztgnk5.s54xkgg3su6yq@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,645 - SL - DEBUG - 18969 - "/work/email_handler.py:1862" - handle_bounce() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - handle bounce for <EmailLog 4>, phase=forward, contact=<Contact 2 c2@example.com 4>, alias=<Alias 4 papist_runner280@sl.local>
2026-07-14 02:15:29,647 - SL - DEBUG - 18969 - "/work/email_handler.py:1870" - handle_bounce() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - User <User 2 Test User user_n8twv718ab@mailbox.test> is not active
2026-07-14 02:15:29,647 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - Finish mail_from <>, rcpt_tos ['sl.lmycyibufqqdemzygmztgnk5.s54xkgg3su6yq@sl.local'], takes 0.007528066635131836 seconds with return code '550 SL E510 so such user'<<===
CELL FWD SIGNED VALID bounce INACTIVE u2.id=2 el=4 is_active=False
     handle_DATA() = '550 SL E510 so such user'
2026-07-14 02:15:29,648 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - c0c5937a-51a7-4891-afdf-2fd59a99d6d7 - set message_id f430f59f-c4b1-41d1-9504-ae19b26689e3
2026-07-14 02:15:29,648 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,648 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - New message, mail from <>, rctp tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local']
2026-07-14 02:15:29,649 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,649 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,650 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,652 - SL - DEBUG - 18969 - "/work/email_handler.py:1824" - handle_transactional_bounce() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - handle transactional bounce sent to sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local
2026-07-14 02:15:29,653 - SL - INFO - 18969 - "/work/email_handler.py:1834" - handle_transactional_bounce() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - No transactional record for <> -> ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local']
2026-07-14 02:15:29,653 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - Finish mail_from <>, rcpt_tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], takes 0.00484156608581543 seconds with return code '250 SL E205 bounce handled'<<===
CELL TXN bounce                     handle_DATA() = '250 SL E205 bounce handled'
2026-07-14 02:15:29,653 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - f430f59f-c4b1-41d1-9504-ae19b26689e3 - set message_id f13069bc-f2cf-448c-bd6d-12eff81702ea
2026-07-14 02:15:29,653 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - f13069bc-f2cf-448c-bd6d-12eff81702ea - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,653 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - f13069bc-f2cf-448c-bd6d-12eff81702ea - New message, mail from user@corp.example, rctp tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local']
2026-07-14 02:15:29,654 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - f13069bc-f2cf-448c-bd6d-12eff81702ea - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,654 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - f13069bc-f2cf-448c-bd6d-12eff81702ea - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,655 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - f13069bc-f2cf-448c-bd6d-12eff81702ea - ==>> Handle mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], header_from:user@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,657 - SL - DEBUG - 18969 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - f13069bc-f2cf-448c-bd6d-12eff81702ea - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:15:29,657 - SL - DEBUG - 18969 - "/work/email_handler.py:2049" - handle() - f13069bc-f2cf-448c-bd6d-12eff81702ea - Ignore out-of-office for transactional emails. Headers: <bound method Message.items of <email.message.Message object at 0x7ad477545360>>
2026-07-14 02:15:29,657 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - f13069bc-f2cf-448c-bd6d-12eff81702ea - Finish mail_from user@corp.example, rcpt_tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], takes 0.0034704208374023438 seconds with return code '250 SL E206 Out of office'<<===
CELL TXN out-of-office              handle_DATA() = '250 SL E206 Out of office'
2026-07-14 02:15:29,658 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - f13069bc-f2cf-448c-bd6d-12eff81702ea - set message_id 00a4c495-88c0-4eb1-adb5-99b8e3af0b05
2026-07-14 02:15:29,658 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,658 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - New message, mail from user@corp.example, rctp tos ['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local']
2026-07-14 02:15:29,658 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,658 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,659 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - ==>> Handle mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,661 - SL - WARNING - 18969 - "/work/email_handler.py:2309" - handle_DATA() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - email handling fail with error:VERPTransactional  mail_from:user@corp.example, rcpt_tos:['sl.lmzcyibrfqqdemzygmztgnk5.qariovyedfplq@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
CELL TXN non-bounce                 handle_DATA() = '250 SL E213 Unknown email ignored'
2026-07-14 02:15:29,918 - SL - INFO - 18969 - "/work/app/events/event_dispatcher.py:62" - send_event() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:29,928 - SL - DEBUG - 18969 - "/work/app/models.py:1459" - generate_random_alias_email() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - generate email islets_blimey058@sl.local
2026-07-14 02:15:29,936 - SL - INFO - 18969 - "/work/app/events/event_dispatcher.py:62" - send_event() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:15:29,951 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - 00a4c495-88c0-4eb1-adb5-99b8e3af0b05 - set message_id ca039aca-1f15-4859-9099-4b17de76dd89
2026-07-14 02:15:29,951 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - ca039aca-1f15-4859-9099-4b17de76dd89 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:29,951 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - ca039aca-1f15-4859-9099-4b17de76dd89 - New message, mail from bounce+5+@sl.local, rctp tos ['islets_blimey058@sl.local']
2026-07-14 02:15:29,952 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - ca039aca-1f15-4859-9099-4b17de76dd89 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:29,952 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - ca039aca-1f15-4859-9099-4b17de76dd89 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:29,952 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - ca039aca-1f15-4859-9099-4b17de76dd89 - ==>> Handle mail_from:bounce+5+@sl.local, rcpt_tos:['islets_blimey058@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:29,962 - SL - WARNING - 18969 - "/work/email_handler.py:2110" - handle() - ca039aca-1f15-4859-9099-4b17de76dd89 - iCloud bounces <EmailLog 5> <Alias 6 islets_blimey058@sl.local>, saved to
2026-07-14 02:15:29,963 - SL - DEBUG - 18969 - "/work/email_handler.py:1862" - handle_bounce() - ca039aca-1f15-4859-9099-4b17de76dd89 - handle bounce for <EmailLog 5>, phase=forward, contact=<Contact 3 c3@example.com 6>, alias=<Alias 6 islets_blimey058@sl.local>
2026-07-14 02:15:29,965 - SL - ERROR - 18969 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - ca039aca-1f15-4859-9099-4b17de76dd89 - Use <Alias 6 islets_blimey058@sl.local> default mailbox <Mailbox 3 user_42jb247xbb@mailbox.test>
NoneType: None
2026-07-14 02:15:29,965 - SL - WARNING - 18969 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - ca039aca-1f15-4859-9099-4b17de76dd89 - cannot get bounce info, debug at
2026-07-14 02:15:29,967 - SL - DEBUG - 18969 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - ca039aca-1f15-4859-9099-4b17de76dd89 - Handle forward bounce <Contact 3 c3@example.com 6> -> <Alias 6 islets_blimey058@sl.local> -> <Mailbox 3 user_42jb247xbb@mailbox.test>. <EmailLog 5>
2026-07-14 02:15:29,970 - SL - WARNING - 18969 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - ca039aca-1f15-4859-9099-4b17de76dd89 - Cannot parse original message from bounce message <Alias 6 islets_blimey058@sl.local> <User 3 Test User user_42jb247xbb@mailbox.test> <Contact 3 c3@example.com 6> refused-emails/full-8bbf8c6d-0f3a-4701-8d54-7d171759e2e2.eml
2026-07-14 02:15:29,973 - SL - DEBUG - 18969 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - ca039aca-1f15-4859-9099-4b17de76dd89 - Create refused email <Refused Email 3 None 2026-07-21T02:15:29.972674+00:00>
2026-07-14 02:15:29,981 - SL - DEBUG - 18969 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - ca039aca-1f15-4859-9099-4b17de76dd89 - Inform user <User 3 Test User user_42jb247xbb@mailbox.test> about a bounce from contact <Contact 3 c3@example.com 6> to alias <Alias 6 islets_blimey058@sl.local>
2026-07-14 02:15:30,007 - SL - DEBUG - 18969 - "/work/app/email_utils.py:303" - send_email() - ca039aca-1f15-4859-9099-4b17de76dd89 - send email to user_42jb247xbb@mailbox.test, subject 'An email sent to islets_blimey058@sl.local cannot be delivered to your mailbox'
2026-07-14 02:15:30,011 - SL - DEBUG - 18969 - "/work/app/mail_sender.py:131" - send() - ca039aca-1f15-4859-9099-4b17de76dd89 - send email with subject 'An email sent to islets_blimey058@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_42jb247xbb@mailbox.test'
2026-07-14 02:15:30,011 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - ca039aca-1f15-4859-9099-4b17de76dd89 - Finish mail_from bounce+5+@sl.local, rcpt_tos ['islets_blimey058@sl.local'], takes 0.060340166091918945 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
CELL iCloud OLD VALID mail_from     mail_from='bounce+5+@sl.local' rcpt='islets_blimey058@sl.local'
     handle_DATA() = '250 SL E211 Bounce Forward phase handled'
2026-07-14 02:15:30,012 - SL - DEBUG - 18969 - "/work/app/log.py:24" - set_message_id() - ca039aca-1f15-4859-9099-4b17de76dd89 - set message_id 0eb80655-3af2-4929-ba18-a92fc62dbe16
2026-07-14 02:15:30,012 - SL - DEBUG - 18969 - "/work/email_handler.py:2342" - _handle() - 0eb80655-3af2-4929-ba18-a92fc62dbe16 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:15:30,012 - SL - INFO - 18969 - "/work/email_handler.py:2343" - _handle() - 0eb80655-3af2-4929-ba18-a92fc62dbe16 - New message, mail from bounce+99999999999999+@sl.local, rctp tos ['islets_blimey058@sl.local']
2026-07-14 02:15:30,012 - SL - INFO - 18969 - "/work/email_handler.py:1956" - handle() - 0eb80655-3af2-4929-ba18-a92fc62dbe16 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:15:30,013 - SL - DEBUG - 18969 - "/work/email_handler.py:1963" - handle() - 0eb80655-3af2-4929-ba18-a92fc62dbe16 - Cannot parse Postfix queue ID from None None
2026-07-14 02:15:30,013 - SL - DEBUG - 18969 - "/work/email_handler.py:1980" - handle() - 0eb80655-3af2-4929-ba18-a92fc62dbe16 - ==>> Handle mail_from:bounce+99999999999999+@sl.local, rcpt_tos:['islets_blimey058@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:15:30,023 - SL - WARNING - 18969 - "/work/email_handler.py:2110" - handle() - 0eb80655-3af2-4929-ba18-a92fc62dbe16 - iCloud bounces None <Alias 6 islets_blimey058@sl.local>, saved to
2026-07-14 02:15:30,023 - SL - WARNING - 18969 - "/work/email_handler.py:1857" - handle_bounce() - 0eb80655-3af2-4929-ba18-a92fc62dbe16 - No such email log
2026-07-14 02:15:30,023 - SL - INFO - 18969 - "/work/email_handler.py:2367" - _handle() - 0eb80655-3af2-4929-ba18-a92fc62dbe16 - Finish mail_from bounce+99999999999999+@sl.local, rcpt_tos ['islets_blimey058@sl.local'], takes 0.011212825775146484 seconds with return code '550 SL E512 No such email log'<<===
CELL iCloud OLD INVALID mail_from   mail_from='bounce+99999999999999+@sl.local' rcpt='islets_blimey058@sl.local'
     handle_DATA() = '550 SL E512 No such email log'

===== CANONICAL OUTCOME SUMMARY (handle_DATA wire responses) =====
FWD OLD  INVALID non-bounce        => 550 SL E512 No such email log
FWD OLD  VALID   non-bounce        => 250 SL E213 Unknown email ignored
FWD OLD  VALID   bounce            => 250 SL E211 Bounce Forward phase handled
FWD SIGNED INVALID non-bounce      => 550 SL E512 No such email log
FWD SIGNED VALID   non-bounce      => 250 SL E213 Unknown email ignored
REP OLD  INVALID bounce            => 550 SL E512 No such email log
REP SIGNED VALID   bounce          => 250 SL E212 Bounce Reply phase handled
REP SIGNED VALID   non-bounce      => 250 SL E213 Unknown email ignored
FWD SIGNED VALID bounce INACTIVE   => 550 SL E510 so such user
TXN bounce                         => 250 SL E205 bounce handled
TXN out-of-office                  => 250 SL E206 Out of office
TXN non-bounce                     => 250 SL E213 Unknown email ignored
iCloud OLD VALID mail_from         => 250 SL E211 Bounce Forward phase handled
iCloud OLD INVALID mail_from       => 550 SL E512 No such email log
===== END SUMMARY (14 cells) =====
OBS_MATRIX_DONE
```

**Executable cross-run / cross-engine comparison.** A standalone program extracts the 14-line canonical outcome-summary block from each captured transcript, hashes it, and checks equality. Source (`compare_runs.py`):

```python
#!/usr/bin/env python3
import re, sys, hashlib

def extract_outcomes(path):
    """Extract the canonical outcome summary block (the 14 '=>' lines) verbatim."""
    out = []
    in_summary = False
    with open(path) as f:
        for line in f:
            if "CANONICAL OUTCOME SUMMARY" in line:
                in_summary = True
                continue
            if "END SUMMARY" in line:
                in_summary = False
                continue
            if in_summary and "=>" in line:
                out.append(line.rstrip("\n"))
    return out

runs = {
    "PG13 run1": "pg13_run1.out",
    "PG13 run2": "pg13_run2.out",
    "PG15 run1": "pg15_run1.out",
    "PG15 run2": "pg15_run2.out",
}
outcomes = {name: extract_outcomes(p) for name, p in runs.items()}

for name, oc in outcomes.items():
    joined = "\n".join(oc)
    h = hashlib.sha256(joined.encode()).hexdigest()
    print(f"{name}: {len(oc)} cells, sha256(outcome-block)={h}")

print()
base = outcomes["PG13 run1"]
all_identical = True
for name, oc in outcomes.items():
    identical = (oc == base)
    all_identical = all_identical and identical
    print(f"{name} == PG13 run1 ? {identical}")

print()
print("ALL FOUR RUNS IDENTICAL:", all_identical)
```

Actual output (`compare_runs.out`):

```text
PG13 run1: 14 cells, sha256(outcome-block)=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354
PG13 run2: 14 cells, sha256(outcome-block)=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354
PG15 run1: 14 cells, sha256(outcome-block)=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354
PG15 run2: 14 cells, sha256(outcome-block)=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354

PG13 run1 == PG13 run1 ? True
PG13 run2 == PG13 run1 ? True
PG15 run1 == PG13 run1 ? True
PG15 run2 == PG13 run1 ? True

ALL FOUR RUNS IDENTICAL: True
```

**Reading of the comparison (observed, not inferred):** the canonical outcome block is **byte-identical across all four runs** — PostgreSQL 13 run 1, PostgreSQL 13 run 2, PostgreSQL 15 run 1, PostgreSQL 15 run 2 — every one hashing to `sha256=6d440cc56adb1964083d81af8d58e5db5eec55abcce9eac8405a51b9e157c354`. Two nuances, reported exactly:

- **Run-to-run (PostgreSQL 13):** run 1 and run 2 are identical even in the concrete `email_log` id values (`valid_fwd=(1,2) valid_reply=3`), because each run began from a **pristine** PostgreSQL 13 schema (`email_log` empty), so auto-increment ids restart at 1.
- **Cross-engine (PostgreSQL 13 vs 15):** the **status strings are identical**, but the concrete probe ids differ (PostgreSQL 13 pristine: `bounce+1+`, `bounce+2+`, `bounce+5+`; PostgreSQL 15 accumulated: `bounce+339+`, `bounce+340+`, `bounce+343+`), because the in-image PostgreSQL 15 `test` database already contained rows from prior suite runs. This is itself confirmation that **the oracle is id-value-independent**: whatever the numeric id, a *valid* `email_log` id yields `E211`/`E212`/`E213` and an *invalid* id yields `E512` — exactly the enumeration signal under investigation.

**Canonical observation-script source (`obs_matrix.py`), reproduced in full:**

```python
"""OBSERVATION SCRIPT (temporary, non-repository) — CANONICAL routing matrix through the
real inbound path MailHandler.handle_DATA() -> _handle() -> handle().
Runnable against either PostgreSQL 15 or PostgreSQL 13 (whichever DB_URI resolves to via the
CONFIG=/work/tests/test.env load_dotenv(override=False) contract). Prints (1) DB env facts,
(2) a canonical handle_DATA outcome for every matrix cell, and (3) a compact OUTCOME SUMMARY
block listing CELL => exact SL status string, for direct PG13-vs-PG15 / run1-vs-run2 comparison."""
import os, asyncio, arrow, sys, subprocess
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from aiosmtpd.smtp import Envelope
import aiosmtpd
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email
from tests.utils import create_new_user

NONBOUNCE = ("From: attacker@evil.example\r\nTo: probe@sl.local\r\nSubject: hi\r\n"
             "Content-Type: text/plain\r\n\r\nhello\r\n")
BOUNCE = ("From: MAILER-DAEMON@evil.example\r\nTo: probe@sl.local\r\nSubject: bounce\r\n"
          "Content-Type: multipart/report; report-type=delivery-status; boundary=\"b\"\r\n\r\n"
          "--b\r\nContent-Type: text/plain\r\n\r\nDelivery failed\r\n"
          "--b\r\nContent-Type: message/delivery-status\r\n\r\nStatus: 5.1.1\r\n--b--\r\n")
# out-of-office: Auto-Submitted header, NOT a bounce (text/plain, non-<> sender)
OOO = ("From: user@corp.example\r\nTo: probe@sl.local\r\nSubject: Away\r\n"
       "Auto-Submitted: auto-replied\r\nContent-Type: text/plain\r\n\r\nI am on holiday\r\n")

def run_handle_data(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    env.original_content = raw.encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

def old(elid): return "%s%s%s" % (config.BOUNCE_PREFIX, elid, config.BOUNCE_SUFFIX)
def oldrep(elid): return "%s+%s+@%s" % (config.BOUNCE_PREFIX_FOR_REPLY_PHASE, elid, config.EMAIL_DOMAIN)

# --- DB environment facts (queried through the app's own engine) ---
pgver = Session.execute("SELECT version()").scalar()
head = Session.execute("SELECT version_num FROM alembic_version").scalar()
ntables = Session.execute("SELECT count(*) FROM information_schema.tables WHERE table_schema='public'").scalar()
print("=== DB ENV FACTS ===")
print("DB_URI(host:port/db) = %s" % config.DB_URI.split("@")[-1])
print("SELECT version()     = %s" % pgver)
print("alembic head         = %s" % head)
print("public table count   = %s" % ntables)
print("python               = %s" % sys.version.split()[0])
print("aiosmtpd             = %s" % aiosmtpd.__version__)
print("EMAIL_DOMAIN         = %s" % config.EMAIL_DOMAIN)
print("=== END ENV FACTS ===\n")

summary = []
transaction = connection.begin()
with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    try:
        user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
        contact = Contact.create(user_id=user.id, alias_id=alias.id,
            website_email="contact@example.com", reply_email="rep@sl.local", commit=True)
        INV = 99999999999999
        def fresh_fwd():
            return EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=False, commit=True).id
        def fresh_rep():
            return EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=True, commit=True).id
        F1, F2 = fresh_fwd(), fresh_fwd()
        R1 = fresh_rep()
        print("FIXTURES user.id=%s alias=%s valid_fwd=(%s,%s) valid_reply=%s INVALID=%s\n"
              % (user.id, alias.email, F1, F2, R1, INV))
        cells = [
          ("FWD OLD  INVALID non-bounce", "attacker@evil.example", old(INV), NONBOUNCE),
          ("FWD OLD  VALID   non-bounce", "attacker@evil.example", old(F1), NONBOUNCE),
          ("FWD OLD  VALID   bounce",     "<>", old(F2), BOUNCE),
          ("FWD SIGNED INVALID non-bounce", "attacker@evil.example", generate_verp_email(VerpType.bounce_forward, INV), NONBOUNCE),
          ("FWD SIGNED VALID   non-bounce", "attacker@evil.example", generate_verp_email(VerpType.bounce_forward, F1), NONBOUNCE),
          ("REP OLD  INVALID bounce",     "<>", oldrep(INV), BOUNCE),
          ("REP SIGNED VALID   bounce",   "<>", generate_verp_email(VerpType.bounce_reply, R1), BOUNCE),
          ("REP SIGNED VALID   non-bounce","attacker@evil.example", generate_verp_email(VerpType.bounce_reply, R1), NONBOUNCE),
        ]
        for label, mf, rcpt, raw in cells:
            r = run_handle_data(mf, rcpt, raw)
            print("CELL %-30s mail_from=%-22r rcpt=%r\n     handle_DATA() = %r" % (label, mf, rcpt, r))
            summary.append((label, r))

        # inactive-user forward bounce -> E510
        u2 = create_new_user(); a2 = Alias.create_new_random(u2); Session.commit()
        c2 = Contact.create(user_id=u2.id, alias_id=a2.id, website_email="c2@example.com", reply_email="rep2@sl.local", commit=True)
        el2 = EmailLog.create(user_id=u2.id, contact_id=c2.id, alias_id=a2.id, is_reply=False, commit=True)
        u2.delete_on = arrow.now().shift(days=1); Session.commit()
        r = run_handle_data("<>", generate_verp_email(VerpType.bounce_forward, el2.id), BOUNCE)
        print("CELL %-30s u2.id=%s el=%s is_active=%s\n     handle_DATA() = %r" % ("FWD SIGNED VALID bounce INACTIVE", u2.id, el2.id, u2.is_active(), r))
        summary.append(("FWD SIGNED VALID bounce INACTIVE", r))

        # transactional: bounce->E205, OOO->E206, non-bounce->VERPTransactional->E213
        taddr = generate_verp_email(VerpType.transactional, 1)
        for lbl, mf, raw in [("TXN bounce", "<>", BOUNCE), ("TXN out-of-office", "user@corp.example", OOO), ("TXN non-bounce", "user@corp.example", NONBOUNCE)]:
            r = run_handle_data(mf, taddr, raw)
            print("CELL %-30s handle_DATA() = %r" % (lbl, r)); summary.append((lbl, r))

        # iCloud branch via handle_DATA: OLD bounce addr in mail_from, alias in rcpt, non-bounce body (no is_bounce gate)
        u3 = create_new_user(); a3 = Alias.create_new_random(u3); Session.commit()
        c3 = Contact.create(user_id=u3.id, alias_id=a3.id, website_email="c3@example.com", reply_email="rep3@sl.local", commit=True)
        el3 = EmailLog.create(user_id=u3.id, contact_id=c3.id, alias_id=a3.id, is_reply=False, commit=True)
        a3_email = a3.email; el3_id = el3.id  # capture ORM-derived values BEFORE handle_DATA mutates/detaches the session
        for lbl, elid in [("iCloud OLD VALID mail_from", el3_id), ("iCloud OLD INVALID mail_from", INV)]:
            r = run_handle_data(old(elid), a3_email, NONBOUNCE)
            print("CELL %-30s mail_from=%r rcpt=%r\n     handle_DATA() = %r" % (lbl, old(elid), a3_email, r)); summary.append((lbl, r))
    finally:
        transaction.rollback(); Session.rollback(); Session.close()

print("\n===== CANONICAL OUTCOME SUMMARY (handle_DATA wire responses) =====")
for lbl, r in summary:
    print("%-34s => %s" % (lbl, r))
print("===== END SUMMARY (%d cells) =====" % len(summary))
print("OBS_MATRIX_DONE")
```

### K.2 Script A — VERP helper round-trips (NON-CANONICAL, Objective 1)

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_A_helpers.py
```

**Observation-script source (`obs_A_helpers.py`):**

```python
"""OBSERVATION SCRIPT A — Objective 1: the two address formats via their generator/parser HELPERS.
NON-CANONICAL: these are direct helper calls (app/email_utils.py), NOT the inbound handle() path.
They show the FORMAT/round-trip only; canonical routing is exercised in script B."""
import os
os.environ["CONFIG"] = "/work/tests/test.env"
from server import create_app
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
from app import config
from app.models import VerpType
from app.email_utils import parse_id_from_bounce, generate_verp_email, get_verp_info_from_email

print("===== CONFIG CONSTANTS (app/config.py) =====")
print("BOUNCE_PREFIX=%r  BOUNCE_SUFFIX=%r" % (config.BOUNCE_PREFIX, config.BOUNCE_SUFFIX))
print("BOUNCE_PREFIX_FOR_REPLY_PHASE=%r" % (config.BOUNCE_PREFIX_FOR_REPLY_PHASE,))
print("TRANSACTIONAL_BOUNCE_PREFIX=%r  TRANSACTIONAL_BOUNCE_SUFFIX=%r" %
      (config.TRANSACTIONAL_BOUNCE_PREFIX, config.TRANSACTIONAL_BOUNCE_SUFFIX))
print("VERP_PREFIX=%r  VERP_MESSAGE_LIFETIME=%r sec (%s days)" %
      (config.VERP_PREFIX, config.VERP_MESSAGE_LIFETIME, config.VERP_MESSAGE_LIFETIME/86400))
print("EMAIL_DOMAIN=%r" % (config.EMAIL_DOMAIN,))

with app.app_context():
    print("\n===== OLD unsigned format: parse_id_from_bounce() [app/email_utils.py:L1258-L1259] (NON-CANONICAL helper) =====")
    for addr in ["bounce+12345+@sl.local", "bounce+369+@sl.local", "bounce+99999999999999+@sl.local"]:
        print("parse_id_from_bounce(%r) = %r" % (addr, parse_id_from_bounce(addr)))

    print("\n===== NEW signed format: generate_verp_email()/get_verp_info_from_email() [app/email_utils.py:L1438-L1498] (NON-CANONICAL helper) =====")
    print("(get_verp_info_from_email returns a 2-tuple (VerpType, object_id); timestamp is validated but not returned)")
    for vt in (VerpType.bounce_forward, VerpType.bounce_reply, VerpType.transactional):
        addr = generate_verp_email(vt, 369)
        info = get_verp_info_from_email(addr)
        print("generate_verp_email(%s, 369) = %r" % (vt.name, addr))
        print("   get_verp_info_from_email(...) = %r  ->  VerpType=%s, object_id=%s"
              % (info, info[0].name if info else None, info[1] if info else None))
print("\nOBS_A_DONE")
```

**Complete verbatim transcript (`obs_A.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/gsimckeuigpyrleilwys
Upload files to local dir
>>> init logging <<<
2026-07-13 18:20:06,470 - SL - DEBUG - 2748 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
===== CONFIG CONSTANTS (app/config.py) =====
BOUNCE_PREFIX='bounce+'  BOUNCE_SUFFIX='+@sl.local'
BOUNCE_PREFIX_FOR_REPLY_PHASE='bounce_reply'
TRANSACTIONAL_BOUNCE_PREFIX='transactional+'  TRANSACTIONAL_BOUNCE_SUFFIX='+@sl.local'
VERP_PREFIX='sl'  VERP_MESSAGE_LIFETIME=432000 sec (5.0 days)
EMAIL_DOMAIN='sl.local'

===== OLD unsigned format: parse_id_from_bounce() [app/email_utils.py:L1258-L1259] (NON-CANONICAL helper) =====
parse_id_from_bounce('bounce+12345+@sl.local') = 12345
parse_id_from_bounce('bounce+369+@sl.local') = 369
parse_id_from_bounce('bounce+99999999999999+@sl.local') = 99999999999999

===== NEW signed format: generate_verp_email()/get_verp_info_from_email() [app/email_utils.py:L1438-L1498] (NON-CANONICAL helper) =====
(get_verp_info_from_email returns a 2-tuple (VerpType, object_id); timestamp is validated but not returned)
generate_verp_email(bounce_forward, 369) = 'sl.lmycyibtgy4syibsgm4deobwgboq.b64xnu7ebjnwm@sl.local'
   get_verp_info_from_email(...) = (<VerpType.bounce_forward: 0>, 369)  ->  VerpType=bounce_forward, object_id=369
generate_verp_email(bounce_reply, 369) = 'sl.lmysyibtgy4syibsgm4deobwgboq.rnnobmm6d2vu6@sl.local'
   get_verp_info_from_email(...) = (<VerpType.bounce_reply: 1>, 369)  ->  VerpType=bounce_reply, object_id=369
generate_verp_email(transactional, 369) = 'sl.lmzcyibtgy4syibsgm4deobwgboq.mn6hurg5rweoc@sl.local'
   get_verp_info_from_email(...) = (<VerpType.transactional: 2>, 369)  ->  VerpType=transactional, object_id=369

OBS_A_DONE
```

### K.3 Script B — canonical forward/reply/signed/inactive matrix

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_B_matrix.py
```

**Observation-script source (`obs_B_matrix.py`):**

```python
"""OBSERVATION SCRIPT B — CANONICAL routing matrix through the real inbound path.
Drives module-level handle() (raw routing outcome) AND MailHandler.handle_DATA() (the wire
response, incl. VERP*->E213 / Exception->E404 mapping). Forward + reply branches
x {valid,invalid id} x {non-bounce, bounce}, plus inactive-user E510 (fresh fixtures)."""
import os, asyncio, arrow
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from aiosmtpd.smtp import Envelope
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email
from tests.utils import create_new_user

NONBOUNCE = ("From: attacker@evil.example\r\nTo: probe@sl.local\r\nSubject: hi\r\n"
             "Content-Type: text/plain\r\n\r\nhello\r\n")
BOUNCE = ("From: MAILER-DAEMON@evil.example\r\nTo: probe@sl.local\r\nSubject: bounce\r\n"
          "Content-Type: multipart/report; report-type=delivery-status; boundary=\"b\"\r\n\r\n"
          "--b\r\nContent-Type: text/plain\r\n\r\nDelivery failed\r\n"
          "--b\r\nContent-Type: message/delivery-status\r\n\r\nStatus: 5.1.1\r\n--b--\r\n")

def run_handle(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    msg = email.message_from_string(raw)
    try:
        return "RETURNED %r" % (email_handler.handle(env, msg),)
    except Exception as e:
        return "RAISED %s(%s)" % (type(e).__name__, e)

def run_handle_data(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    env.original_content = raw.encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

def old(elid): return "%s%s%s" % (config.BOUNCE_PREFIX, elid, config.BOUNCE_SUFFIX)
def oldrep(elid): return "%s+%s+@%s" % (config.BOUNCE_PREFIX_FOR_REPLY_PHASE, elid, config.EMAIL_DOMAIN)

transaction = connection.begin()
with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    try:
        user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
        contact = Contact.create(user_id=user.id, alias_id=alias.id,
            website_email="contact@example.com", reply_email="rep@sl.local", commit=True)
        INV = 99999999999999
        # a FRESH forward email_log per bounce cell (so a prior bounce does not perturb the next)
        def fresh_fwd():
            el = EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=False, commit=True); return el.id
        def fresh_rep():
            el = EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=True, commit=True); return el.id
        VID_F1, VID_F2, VID_F3 = fresh_fwd(), fresh_fwd(), fresh_fwd()
        VID_R1, VID_R2 = fresh_rep(), fresh_rep()
        print("FIXTURES: user.id=%s alias=%s  fwd_ids=%s  reply_ids=%s  INVALID_id=%s"
              % (user.id, alias.email, (VID_F1,VID_F2,VID_F3), (VID_R1,VID_R2), INV))
        cells = [
          ("FWD OLD-unsigned INVALID non-bounce", "attacker@evil.example", old(INV), NONBOUNCE),
          ("FWD OLD-unsigned VALID   non-bounce", "attacker@evil.example", old(VID_F1), NONBOUNCE),
          ("FWD OLD-unsigned INVALID bounce",     "<>", old(INV), BOUNCE),
          ("FWD OLD-unsigned VALID   bounce",     "<>", old(VID_F2), BOUNCE),
          ("FWD SIGNED       INVALID non-bounce", "attacker@evil.example", generate_verp_email(VerpType.bounce_forward, INV), NONBOUNCE),
          ("FWD SIGNED       VALID   non-bounce", "attacker@evil.example", generate_verp_email(VerpType.bounce_forward, VID_F1), NONBOUNCE),
          ("FWD SIGNED       VALID   bounce",     "<>", generate_verp_email(VerpType.bounce_forward, VID_F3), BOUNCE),
          ("REP OLD-unsigned INVALID bounce",     "<>", oldrep(INV), BOUNCE),
          ("REP SIGNED       INVALID non-bounce", "attacker@evil.example", generate_verp_email(VerpType.bounce_reply, INV), NONBOUNCE),
          ("REP SIGNED       VALID   bounce",     "<>", generate_verp_email(VerpType.bounce_reply, VID_R1), BOUNCE),
          ("REP SIGNED       VALID   non-bounce", "attacker@evil.example", generate_verp_email(VerpType.bounce_reply, VID_R2), NONBOUNCE),
        ]
        for label, mf, rcpt, raw in cells:
            print("\n########## CELL: %s ##########" % label)
            print("ENVELOPE mail_from=%r rcpt_to=%r" % (mf, rcpt))
            print(">>> handle()      : %s" % run_handle(mf, rcpt, raw))
            print(">>> handle_DATA() : %r" % (run_handle_data(mf, rcpt, raw),))

        print("\n########## CELL: FWD SIGNED VALID bounce, INACTIVE user -> E510 (FRESH fixtures) ##########")
        u2 = create_new_user(); a2 = Alias.create_new_random(u2); Session.commit()
        c2 = Contact.create(user_id=u2.id, alias_id=a2.id, website_email="c2@example.com", reply_email="rep2@sl.local", commit=True)
        el2 = EmailLog.create(user_id=u2.id, contact_id=c2.id, alias_id=a2.id, is_reply=False, commit=True)
        u2.delete_on = arrow.now().shift(days=1); Session.commit()
        print("u2.id=%s email_log_id=%s  u2.is_active()=%s (delete_on=%r)" % (u2.id, el2.id, u2.is_active(), u2.delete_on))
        rcpt = generate_verp_email(VerpType.bounce_forward, el2.id)
        print("ENVELOPE mail_from='<>' rcpt_to=%r" % rcpt)
        print(">>> handle()      : %s" % run_handle("<>", rcpt, BOUNCE))
        print(">>> handle_DATA() : %r" % (run_handle_data("<>", rcpt, BOUNCE),))
    finally:
        transaction.rollback(); Session.rollback(); Session.close()
print("\nOBS_B_DONE")
```

**Complete verbatim transcript (`obs_B.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/zjdauaniqonxlzdwzlrd
Upload files to local dir
>>> init logging <<<
2026-07-13 18:20:08,847 - SL - DEBUG - 2769 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-13 18:20:09,938 - SL - DEBUG - 2769 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 18:20:09,939 - SL - DEBUG - 2769 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 18:20:09,940 - SL - DEBUG - 2769 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:20:09,940 - SL - DEBUG - 2769 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:20:10,241 - SL - INFO - 2769 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:20:10,254 - SL - DEBUG - 2769 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email dandle_rifted086@sl.local
2026-07-13 18:20:10,261 - SL - INFO - 2769 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURES: user.id=479 alias=dandle_rifted086@sl.local  fwd_ids=(385, 386, 387)  reply_ids=(388, 389)  INVALID_id=99999999999999

########## CELL: FWD OLD-unsigned INVALID non-bounce ##########
ENVELOPE mail_from='attacker@evil.example' rcpt_to='bounce+99999999999999+@sl.local'
2026-07-13 18:20:10,306 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,306 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,308 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,312 - SL - WARNING - 2769 - "/work/email_handler.py:2066" - handle() -  - No such email log
>>> handle()      : RETURNED '550 SL E512 No such email log'
2026-07-13 18:20:10,314 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() -  - set message_id 237f61d8-c27b-41e5-acf6-c96dfe1d2a11
2026-07-13 18:20:10,314 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,314 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:20:10,315 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,315 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,316 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,319 - SL - WARNING - 2769 - "/work/email_handler.py:2066" - handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - No such email log
2026-07-13 18:20:10,320 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.00516200065612793 seconds with return code '550 SL E512 No such email log'<<===
>>> handle_DATA() : '550 SL E512 No such email log'

########## CELL: FWD OLD-unsigned VALID   non-bounce ##########
ENVELOPE mail_from='attacker@evil.example' rcpt_to='bounce+385+@sl.local'
2026-07-13 18:20:10,320 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,320 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,321 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+385+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
>>> handle()      : RAISED VERPForward(VERPForward )
2026-07-13 18:20:10,325 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - 237f61d8-c27b-41e5-acf6-c96dfe1d2a11 - set message_id 33765b7f-53ee-4f03-8352-df8b41898755
2026-07-13 18:20:10,325 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,325 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - New message, mail from attacker@evil.example, rctp tos ['bounce+385+@sl.local']
2026-07-13 18:20:10,325 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,326 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,326 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+385+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,331 - SL - WARNING - 2769 - "/work/email_handler.py:2309" - handle_DATA() - 33765b7f-53ee-4f03-8352-df8b41898755 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+385+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
>>> handle_DATA() : '250 SL E213 Unknown email ignored'

########## CELL: FWD OLD-unsigned INVALID bounce ##########
ENVELOPE mail_from='<>' rcpt_to='bounce+99999999999999+@sl.local'
2026-07-13 18:20:10,331 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,331 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,332 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - ==>> Handle mail_from:<>, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,336 - SL - WARNING - 2769 - "/work/email_handler.py:2066" - handle() - 33765b7f-53ee-4f03-8352-df8b41898755 - No such email log
>>> handle()      : RETURNED '550 SL E512 No such email log'
2026-07-13 18:20:10,336 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - 33765b7f-53ee-4f03-8352-df8b41898755 - set message_id 8f1fe85c-23aa-4b46-be54-599cf0ef5576
2026-07-13 18:20:10,336 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,336 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - New message, mail from <>, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:20:10,337 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,337 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,338 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - ==>> Handle mail_from:<>, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,341 - SL - WARNING - 2769 - "/work/email_handler.py:2066" - handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - No such email log
2026-07-13 18:20:10,341 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Finish mail_from <>, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.0046634674072265625 seconds with return code '550 SL E512 No such email log'<<===
>>> handle_DATA() : '550 SL E512 No such email log'

########## CELL: FWD OLD-unsigned VALID   bounce ##########
ENVELOPE mail_from='<>' rcpt_to='bounce+386+@sl.local'
2026-07-13 18:20:10,341 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,341 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,342 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - ==>> Handle mail_from:<>, rcpt_tos:['bounce+386+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,349 - SL - DEBUG - 2769 - "/work/email_handler.py:1862" - handle_bounce() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - handle bounce for <EmailLog 386>, phase=forward, contact=<Contact 155 contact@example.com 822>, alias=<Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,351 - SL - ERROR - 2769 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Use <Alias 822 dandle_rifted086@sl.local> default mailbox <Mailbox 559 user_jzw80zwu73@mailbox.test>
NoneType: None
2026-07-13 18:20:10,351 - SL - WARNING - 2769 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - cannot get bounce info, debug at
2026-07-13 18:20:10,354 - SL - DEBUG - 2769 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Handle forward bounce <Contact 155 contact@example.com 822> -> <Alias 822 dandle_rifted086@sl.local> -> <Mailbox 559 user_jzw80zwu73@mailbox.test>. <EmailLog 386>
2026-07-13 18:20:10,358 - SL - WARNING - 2769 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Cannot parse original message from bounce message <Alias 822 dandle_rifted086@sl.local> <User 479 Test User user_jzw80zwu73@mailbox.test> <Contact 155 contact@example.com 822> refused-emails/full-2be1c7be-f032-408d-b63a-6a726657ee81.eml
2026-07-13 18:20:10,361 - SL - DEBUG - 2769 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Create refused email <Refused Email 38 None 2026-07-20T18:20:10.360368+00:00>
2026-07-13 18:20:10,370 - SL - DEBUG - 2769 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - Inform user <User 479 Test User user_jzw80zwu73@mailbox.test> about a bounce from contact <Contact 155 contact@example.com 822> to alias <Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,397 - SL - DEBUG - 2769 - "/work/app/email_utils.py:303" - send_email() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - send email to user_jzw80zwu73@mailbox.test, subject 'An email sent to dandle_rifted086@sl.local cannot be delivered to your mailbox'
2026-07-13 18:20:10,402 - SL - DEBUG - 2769 - "/work/app/mail_sender.py:131" - send() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - send email with subject 'An email sent to dandle_rifted086@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_jzw80zwu73@mailbox.test'
>>> handle()      : RETURNED '250 SL E211 Bounce Forward phase handled'
2026-07-13 18:20:10,403 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - 8f1fe85c-23aa-4b46-be54-599cf0ef5576 - set message_id 3313da47-5800-4ee5-b109-8e799da5b90e
2026-07-13 18:20:10,403 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,403 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - New message, mail from <>, rctp tos ['bounce+386+@sl.local']
2026-07-13 18:20:10,403 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,403 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,404 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - ==>> Handle mail_from:<>, rcpt_tos:['bounce+386+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,409 - SL - DEBUG - 2769 - "/work/email_handler.py:1862" - handle_bounce() - 3313da47-5800-4ee5-b109-8e799da5b90e - handle bounce for <EmailLog 386>, phase=forward, contact=<Contact 155 contact@example.com 822>, alias=<Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,410 - SL - ERROR - 2769 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - 3313da47-5800-4ee5-b109-8e799da5b90e - Use <Alias 822 dandle_rifted086@sl.local> default mailbox <Mailbox 559 user_jzw80zwu73@mailbox.test>
NoneType: None
2026-07-13 18:20:10,410 - SL - WARNING - 2769 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - 3313da47-5800-4ee5-b109-8e799da5b90e - cannot get bounce info, debug at
2026-07-13 18:20:10,411 - SL - DEBUG - 2769 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - 3313da47-5800-4ee5-b109-8e799da5b90e - Handle forward bounce <Contact 155 contact@example.com 822> -> <Alias 822 dandle_rifted086@sl.local> -> <Mailbox 559 user_jzw80zwu73@mailbox.test>. <EmailLog 386>
2026-07-13 18:20:10,415 - SL - WARNING - 2769 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - 3313da47-5800-4ee5-b109-8e799da5b90e - Cannot parse original message from bounce message <Alias 822 dandle_rifted086@sl.local> <User 479 Test User user_jzw80zwu73@mailbox.test> <Contact 155 contact@example.com 822> refused-emails/full-5f93788a-cd00-44b7-b2e9-4a6e2d6201fb.eml
2026-07-13 18:20:10,417 - SL - DEBUG - 2769 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - 3313da47-5800-4ee5-b109-8e799da5b90e - Create refused email <Refused Email 39 None 2026-07-20T18:20:10.416757+00:00>
2026-07-13 18:20:10,425 - SL - DEBUG - 2769 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - 3313da47-5800-4ee5-b109-8e799da5b90e - Inform user <User 479 Test User user_jzw80zwu73@mailbox.test> about a bounce from contact <Contact 155 contact@example.com 822> to alias <Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,451 - SL - DEBUG - 2769 - "/work/app/email_utils.py:303" - send_email() - 3313da47-5800-4ee5-b109-8e799da5b90e - send email to user_jzw80zwu73@mailbox.test, subject 'An email sent to dandle_rifted086@sl.local cannot be delivered to your mailbox'
2026-07-13 18:20:10,456 - SL - DEBUG - 2769 - "/work/app/mail_sender.py:131" - send() - 3313da47-5800-4ee5-b109-8e799da5b90e - send email with subject 'An email sent to dandle_rifted086@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_jzw80zwu73@mailbox.test'
2026-07-13 18:20:10,456 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - Finish mail_from <>, rcpt_tos ['bounce+386+@sl.local'], takes 0.053072214126586914 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
>>> handle_DATA() : '250 SL E211 Bounce Forward phase handled'

########## CELL: FWD SIGNED       INVALID non-bounce ##########
ENVELOPE mail_from='attacker@evil.example' rcpt_to='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.d2u7yvh73aezm@sl.local'
2026-07-13 18:20:10,456 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,456 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,457 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.d2u7yvh73aezm@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,461 - SL - WARNING - 2769 - "/work/email_handler.py:2066" - handle() - 3313da47-5800-4ee5-b109-8e799da5b90e - No such email log
>>> handle()      : RETURNED '550 SL E512 No such email log'
2026-07-13 18:20:10,461 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - 3313da47-5800-4ee5-b109-8e799da5b90e - set message_id 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69
2026-07-13 18:20:10,461 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,461 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.d2u7yvh73aezm@sl.local']
2026-07-13 18:20:10,462 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,462 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,463 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.d2u7yvh73aezm@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,466 - SL - WARNING - 2769 - "/work/email_handler.py:2066" - handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - No such email log
2026-07-13 18:20:10,466 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.d2u7yvh73aezm@sl.local'], takes 0.0051538944244384766 seconds with return code '550 SL E512 No such email log'<<===
>>> handle_DATA() : '550 SL E512 No such email log'

########## CELL: FWD SIGNED       VALID   non-bounce ##########
ENVELOPE mail_from='attacker@evil.example' rcpt_to='sl.lmycyibtha2syibsgm4deobwgboq.mkvkzf33xtabo@sl.local'
2026-07-13 18:20:10,467 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,467 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,468 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibtha2syibsgm4deobwgboq.mkvkzf33xtabo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
>>> handle()      : RAISED VERPForward(VERPForward )
2026-07-13 18:20:10,472 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - 9a07d7cd-7076-4e77-b3ca-8cb24a1d9f69 - set message_id b7ffb1cf-4b01-45b9-a8fa-6461028af4e2
2026-07-13 18:20:10,472 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,472 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibtha2syibsgm4deobwgboq.mkvkzf33xtabo@sl.local']
2026-07-13 18:20:10,472 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,472 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,473 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibtha2syibsgm4deobwgboq.mkvkzf33xtabo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,477 - SL - WARNING - 2769 - "/work/email_handler.py:2309" - handle_DATA() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibtha2syibsgm4deobwgboq.mkvkzf33xtabo@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
>>> handle_DATA() : '250 SL E213 Unknown email ignored'

########## CELL: FWD SIGNED       VALID   bounce ##########
ENVELOPE mail_from='<>' rcpt_to='sl.lmycyibtha3syibsgm4deobwgboq.luaymr7wkenmi@sl.local'
2026-07-13 18:20:10,478 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,478 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,479 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibtha3syibsgm4deobwgboq.luaymr7wkenmi@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,484 - SL - DEBUG - 2769 - "/work/email_handler.py:1862" - handle_bounce() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - handle bounce for <EmailLog 387>, phase=forward, contact=<Contact 155 contact@example.com 822>, alias=<Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,485 - SL - ERROR - 2769 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Use <Alias 822 dandle_rifted086@sl.local> default mailbox <Mailbox 559 user_jzw80zwu73@mailbox.test>
NoneType: None
2026-07-13 18:20:10,485 - SL - WARNING - 2769 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - cannot get bounce info, debug at
2026-07-13 18:20:10,486 - SL - DEBUG - 2769 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Handle forward bounce <Contact 155 contact@example.com 822> -> <Alias 822 dandle_rifted086@sl.local> -> <Mailbox 559 user_jzw80zwu73@mailbox.test>. <EmailLog 387>
2026-07-13 18:20:10,490 - SL - WARNING - 2769 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Cannot parse original message from bounce message <Alias 822 dandle_rifted086@sl.local> <User 479 Test User user_jzw80zwu73@mailbox.test> <Contact 155 contact@example.com 822> refused-emails/full-3afc994f-69c9-4346-9bcf-fcba6ff64f34.eml
2026-07-13 18:20:10,493 - SL - DEBUG - 2769 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Create refused email <Refused Email 40 None 2026-07-20T18:20:10.492606+00:00>
2026-07-13 18:20:10,502 - SL - DEBUG - 2769 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - Inform user <User 479 Test User user_jzw80zwu73@mailbox.test> about a bounce from contact <Contact 155 contact@example.com 822> to alias <Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,529 - SL - DEBUG - 2769 - "/work/app/email_utils.py:303" - send_email() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - send email to user_jzw80zwu73@mailbox.test, subject 'An email sent to dandle_rifted086@sl.local cannot be delivered to your mailbox'
2026-07-13 18:20:10,533 - SL - DEBUG - 2769 - "/work/app/mail_sender.py:131" - send() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - send email with subject 'An email sent to dandle_rifted086@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_jzw80zwu73@mailbox.test'
>>> handle()      : RETURNED '250 SL E211 Bounce Forward phase handled'
2026-07-13 18:20:10,534 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - b7ffb1cf-4b01-45b9-a8fa-6461028af4e2 - set message_id 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2
2026-07-13 18:20:10,534 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,534 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - New message, mail from <>, rctp tos ['sl.lmycyibtha3syibsgm4deobwgboq.luaymr7wkenmi@sl.local']
2026-07-13 18:20:10,535 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,535 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,536 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibtha3syibsgm4deobwgboq.luaymr7wkenmi@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,542 - SL - DEBUG - 2769 - "/work/email_handler.py:1862" - handle_bounce() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - handle bounce for <EmailLog 387>, phase=forward, contact=<Contact 155 contact@example.com 822>, alias=<Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,543 - SL - ERROR - 2769 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Use <Alias 822 dandle_rifted086@sl.local> default mailbox <Mailbox 559 user_jzw80zwu73@mailbox.test>
NoneType: None
2026-07-13 18:20:10,543 - SL - WARNING - 2769 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - cannot get bounce info, debug at
2026-07-13 18:20:10,545 - SL - DEBUG - 2769 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Handle forward bounce <Contact 155 contact@example.com 822> -> <Alias 822 dandle_rifted086@sl.local> -> <Mailbox 559 user_jzw80zwu73@mailbox.test>. <EmailLog 387>
2026-07-13 18:20:10,549 - SL - WARNING - 2769 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Cannot parse original message from bounce message <Alias 822 dandle_rifted086@sl.local> <User 479 Test User user_jzw80zwu73@mailbox.test> <Contact 155 contact@example.com 822> refused-emails/full-5e91cec5-fa29-48cb-afd6-c5ef9c64c0d3.eml
2026-07-13 18:20:10,551 - SL - DEBUG - 2769 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Create refused email <Refused Email 41 None 2026-07-20T18:20:10.551262+00:00>
2026-07-13 18:20:10,562 - SL - DEBUG - 2769 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Inform user <User 479 Test User user_jzw80zwu73@mailbox.test> about a bounce from contact <Contact 155 contact@example.com 822> to alias <Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,589 - SL - DEBUG - 2769 - "/work/app/email_utils.py:303" - send_email() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - send email to user_jzw80zwu73@mailbox.test, subject 'An email sent to dandle_rifted086@sl.local cannot be delivered to your mailbox'
2026-07-13 18:20:10,596 - SL - DEBUG - 2769 - "/work/app/mail_sender.py:131" - send() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - send email with subject 'An email sent to dandle_rifted086@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_jzw80zwu73@mailbox.test'
2026-07-13 18:20:10,596 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Finish mail_from <>, rcpt_tos ['sl.lmycyibtha3syibsgm4deobwgboq.luaymr7wkenmi@sl.local'], takes 0.06151533126831055 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
>>> handle_DATA() : '250 SL E211 Bounce Forward phase handled'

########## CELL: REP OLD-unsigned INVALID bounce ##########
ENVELOPE mail_from='<>' rcpt_to='bounce_reply+99999999999999+@sl.local'
2026-07-13 18:20:10,596 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,596 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,597 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - ==>> Handle mail_from:<>, rcpt_tos:['bounce_reply+99999999999999+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,601 - SL - WARNING - 2769 - "/work/email_handler.py:2086" - handle() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - No such email log
>>> handle()      : RETURNED '550 SL E512 No such email log'
2026-07-13 18:20:10,602 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - 033e33c4-fdb3-4ae5-b791-7deacdbdb2b2 - set message_id 4840e32f-70f8-4381-b646-9906b9be8629
2026-07-13 18:20:10,602 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,602 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - New message, mail from <>, rctp tos ['bounce_reply+99999999999999+@sl.local']
2026-07-13 18:20:10,602 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,602 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,603 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - ==>> Handle mail_from:<>, rcpt_tos:['bounce_reply+99999999999999+@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,607 - SL - WARNING - 2769 - "/work/email_handler.py:2086" - handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - No such email log
2026-07-13 18:20:10,607 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - Finish mail_from <>, rcpt_tos ['bounce_reply+99999999999999+@sl.local'], takes 0.005070686340332031 seconds with return code '550 SL E512 No such email log'<<===
>>> handle_DATA() : '550 SL E512 No such email log'

########## CELL: REP SIGNED       INVALID non-bounce ##########
ENVELOPE mail_from='attacker@evil.example' rcpt_to='sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.cidobv3wlys4e@sl.local'
2026-07-13 18:20:10,607 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,607 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,608 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.cidobv3wlys4e@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,611 - SL - WARNING - 2769 - "/work/email_handler.py:2086" - handle() - 4840e32f-70f8-4381-b646-9906b9be8629 - No such email log
>>> handle()      : RETURNED '550 SL E512 No such email log'
2026-07-13 18:20:10,612 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - 4840e32f-70f8-4381-b646-9906b9be8629 - set message_id f526b135-c058-413f-8109-94209d62e747
2026-07-13 18:20:10,612 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - f526b135-c058-413f-8109-94209d62e747 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,612 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - f526b135-c058-413f-8109-94209d62e747 - New message, mail from attacker@evil.example, rctp tos ['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.cidobv3wlys4e@sl.local']
2026-07-13 18:20:10,612 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - f526b135-c058-413f-8109-94209d62e747 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,612 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - f526b135-c058-413f-8109-94209d62e747 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,613 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - f526b135-c058-413f-8109-94209d62e747 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.cidobv3wlys4e@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,616 - SL - WARNING - 2769 - "/work/email_handler.py:2086" - handle() - f526b135-c058-413f-8109-94209d62e747 - No such email log
2026-07-13 18:20:10,616 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - f526b135-c058-413f-8109-94209d62e747 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyyf2.cidobv3wlys4e@sl.local'], takes 0.0046198368072509766 seconds with return code '550 SL E512 No such email log'<<===
>>> handle_DATA() : '550 SL E512 No such email log'

########## CELL: REP SIGNED       VALID   bounce ##########
ENVELOPE mail_from='<>' rcpt_to='sl.lmysyibtha4cyibsgm4deobwgboq.6t6t7b3x2o43g@sl.local'
2026-07-13 18:20:10,617 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - f526b135-c058-413f-8109-94209d62e747 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,617 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - f526b135-c058-413f-8109-94209d62e747 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,618 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - f526b135-c058-413f-8109-94209d62e747 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmysyibtha4cyibsgm4deobwgboq.6t6t7b3x2o43g@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,622 - SL - DEBUG - 2769 - "/work/email_handler.py:1862" - handle_bounce() - f526b135-c058-413f-8109-94209d62e747 - handle bounce for <EmailLog 388>, phase=reply, contact=<Contact 155 contact@example.com 822>, alias=<Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,623 - SL - DEBUG - 2769 - "/work/email_handler.py:1605" - handle_bounce_reply_phase() - f526b135-c058-413f-8109-94209d62e747 - Handle reply bounce <Mailbox 559 user_jzw80zwu73@mailbox.test> -> <Alias 822 dandle_rifted086@sl.local> -> <Contact 155 contact@example.com 822>.<EmailLog 388>
2026-07-13 18:20:10,623 - SL - WARNING - 2769 - "/work/email_handler.py:1615" - handle_bounce_reply_phase() - f526b135-c058-413f-8109-94209d62e747 - cannot get bounce info, debug at
2026-07-13 18:20:10,628 - SL - DEBUG - 2769 - "/work/email_handler.py:1640" - handle_bounce_reply_phase() - f526b135-c058-413f-8109-94209d62e747 - Create refused email <Refused Email 42 None 2026-07-20T18:20:10.627590+00:00>
2026-07-13 18:20:10,633 - SL - DEBUG - 2769 - "/work/email_handler.py:1651" - handle_bounce_reply_phase() - f526b135-c058-413f-8109-94209d62e747 - Inform user <User 479 Test User user_jzw80zwu73@mailbox.test> about bounced email sent by <Alias 822 dandle_rifted086@sl.local> to <Contact 155 contact@example.com 822>
2026-07-13 18:20:10,657 - SL - DEBUG - 2769 - "/work/app/email_utils.py:303" - send_email() - f526b135-c058-413f-8109-94209d62e747 - send email to user_jzw80zwu73@mailbox.test, subject 'Email cannot be sent to contact@example.com from your alias dandle_rifted086@sl.local'
2026-07-13 18:20:10,660 - SL - DEBUG - 2769 - "/work/app/mail_sender.py:131" - send() - f526b135-c058-413f-8109-94209d62e747 - send email with subject 'Email cannot be sent to contact@example.com from your alias dandle_rifted086@sl.local', from '"noreply@sl.local" <noreply@sl.local>' to 'user_jzw80zwu73@mailbox.test'
>>> handle()      : RETURNED '250 SL E212 Bounce Reply phase handled'
2026-07-13 18:20:10,661 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - f526b135-c058-413f-8109-94209d62e747 - set message_id 1027a7f8-9f4d-4ea0-b62b-39b38994279c
2026-07-13 18:20:10,661 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,661 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - New message, mail from <>, rctp tos ['sl.lmysyibtha4cyibsgm4deobwgboq.6t6t7b3x2o43g@sl.local']
2026-07-13 18:20:10,662 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,662 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,663 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmysyibtha4cyibsgm4deobwgboq.6t6t7b3x2o43g@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,668 - SL - DEBUG - 2769 - "/work/email_handler.py:1862" - handle_bounce() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - handle bounce for <EmailLog 388>, phase=reply, contact=<Contact 155 contact@example.com 822>, alias=<Alias 822 dandle_rifted086@sl.local>
2026-07-13 18:20:10,668 - SL - DEBUG - 2769 - "/work/email_handler.py:1605" - handle_bounce_reply_phase() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - Handle reply bounce <Mailbox 559 user_jzw80zwu73@mailbox.test> -> <Alias 822 dandle_rifted086@sl.local> -> <Contact 155 contact@example.com 822>.<EmailLog 388>
2026-07-13 18:20:10,668 - SL - WARNING - 2769 - "/work/email_handler.py:1615" - handle_bounce_reply_phase() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - cannot get bounce info, debug at
2026-07-13 18:20:10,673 - SL - DEBUG - 2769 - "/work/email_handler.py:1640" - handle_bounce_reply_phase() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - Create refused email <Refused Email 43 None 2026-07-20T18:20:10.672399+00:00>
2026-07-13 18:20:10,677 - SL - DEBUG - 2769 - "/work/email_handler.py:1651" - handle_bounce_reply_phase() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - Inform user <User 479 Test User user_jzw80zwu73@mailbox.test> about bounced email sent by <Alias 822 dandle_rifted086@sl.local> to <Contact 155 contact@example.com 822>
2026-07-13 18:20:10,703 - SL - DEBUG - 2769 - "/work/app/email_utils.py:303" - send_email() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - send email to user_jzw80zwu73@mailbox.test, subject 'Email cannot be sent to contact@example.com from your alias dandle_rifted086@sl.local'
2026-07-13 18:20:10,707 - SL - DEBUG - 2769 - "/work/app/mail_sender.py:131" - send() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - send email with subject 'Email cannot be sent to contact@example.com from your alias dandle_rifted086@sl.local', from '"noreply@sl.local" <noreply@sl.local>' to 'user_jzw80zwu73@mailbox.test'
2026-07-13 18:20:10,707 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - Finish mail_from <>, rcpt_tos ['sl.lmysyibtha4cyibsgm4deobwgboq.6t6t7b3x2o43g@sl.local'], takes 0.04617452621459961 seconds with return code '250 SL E212 Bounce Reply phase handled'<<===
>>> handle_DATA() : '250 SL E212 Bounce Reply phase handled'

########## CELL: REP SIGNED       VALID   non-bounce ##########
ENVELOPE mail_from='attacker@evil.example' rcpt_to='sl.lmysyibtha4syibsgm4deobwgboq.4exnsuakl4jf2@sl.local'
2026-07-13 18:20:10,708 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,708 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,709 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtha4syibsgm4deobwgboq.4exnsuakl4jf2@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
>>> handle()      : RAISED VERPReply(VERPReply cannot handle email sent to reply VERP, <Alias 822 dandle_rifted086@sl.local> -> <Contact 155 contact@example.com 822> (<EmailLog 389>, <User 479 Test User user_jzw80zwu73@mailbox.test>)
2026-07-13 18:20:10,717 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - 1027a7f8-9f4d-4ea0-b62b-39b38994279c - set message_id e776679e-6ad6-47a9-8045-89a998d1c3b3
2026-07-13 18:20:10,717 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:10,717 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - New message, mail from attacker@evil.example, rctp tos ['sl.lmysyibtha4syibsgm4deobwgboq.4exnsuakl4jf2@sl.local']
2026-07-13 18:20:10,718 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:10,718 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:10,719 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtha4syibsgm4deobwgboq.4exnsuakl4jf2@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:10,725 - SL - WARNING - 2769 - "/work/email_handler.py:2309" - handle_DATA() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - email handling fail with error:VERPReply cannot handle email sent to reply VERP, <Alias 822 dandle_rifted086@sl.local> -> <Contact 155 contact@example.com 822> (<EmailLog 389>, <User 479 Test User user_jzw80zwu73@mailbox.test> mail_from:attacker@evil.example, rcpt_tos:['sl.lmysyibtha4syibsgm4deobwgboq.4exnsuakl4jf2@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
>>> handle_DATA() : '250 SL E213 Unknown email ignored'

########## CELL: FWD SIGNED VALID bounce, INACTIVE user -> E510 (FRESH fixtures) ##########
2026-07-13 18:20:10,984 - SL - INFO - 2769 - "/work/app/events/event_dispatcher.py:62" - send_event() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:20:10,995 - SL - DEBUG - 2769 - "/work/app/models.py:1459" - generate_random_alias_email() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - generate email pistol_sautes769@sl.local
2026-07-13 18:20:11,002 - SL - INFO - 2769 - "/work/app/events/event_dispatcher.py:62" - send_event() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - Not sending events because webhook is not configured and allowed to be empty
u2.id=480 email_log_id=390  u2.is_active()=False (delete_on=<Arrow [2026-07-14T18:20:11.015646+00:00]>)
ENVELOPE mail_from='<>' rcpt_to='sl.lmycyibtheycyibsgm4deobwgboq.bzqmzgc5lk4ty@sl.local'
2026-07-13 18:20:11,021 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:11,021 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:11,023 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibtheycyibsgm4deobwgboq.bzqmzgc5lk4ty@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:11,028 - SL - DEBUG - 2769 - "/work/email_handler.py:1862" - handle_bounce() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - handle bounce for <EmailLog 390>, phase=forward, contact=<Contact 156 c2@example.com 824>, alias=<Alias 824 pistol_sautes769@sl.local>
2026-07-13 18:20:11,028 - SL - DEBUG - 2769 - "/work/email_handler.py:1870" - handle_bounce() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - User <User 480 Test User user_v9s2znb0s5@mailbox.test> is not active
>>> handle()      : RETURNED '550 SL E510 so such user'
2026-07-13 18:20:11,029 - SL - DEBUG - 2769 - "/work/app/log.py:24" - set_message_id() - e776679e-6ad6-47a9-8045-89a998d1c3b3 - set message_id 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e
2026-07-13 18:20:11,029 - SL - DEBUG - 2769 - "/work/email_handler.py:2342" - _handle() - 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e - ====>=====>====>====>====>====>====>====>
2026-07-13 18:20:11,029 - SL - INFO - 2769 - "/work/email_handler.py:2343" - _handle() - 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e - New message, mail from <>, rctp tos ['sl.lmycyibtheycyibsgm4deobwgboq.bzqmzgc5lk4ty@sl.local']
2026-07-13 18:20:11,029 - SL - INFO - 2769 - "/work/email_handler.py:1956" - handle() - 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:20:11,030 - SL - DEBUG - 2769 - "/work/email_handler.py:1963" - handle() - 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e - Cannot parse Postfix queue ID from None None
2026-07-13 18:20:11,030 - SL - DEBUG - 2769 - "/work/email_handler.py:1980" - handle() - 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibtheycyibsgm4deobwgboq.bzqmzgc5lk4ty@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:20:11,033 - SL - DEBUG - 2769 - "/work/email_handler.py:1862" - handle_bounce() - 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e - handle bounce for <EmailLog 390>, phase=forward, contact=<Contact 156 c2@example.com 824>, alias=<Alias 824 pistol_sautes769@sl.local>
2026-07-13 18:20:11,033 - SL - DEBUG - 2769 - "/work/email_handler.py:1870" - handle_bounce() - 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e - User <User 480 Test User user_v9s2znb0s5@mailbox.test> is not active
2026-07-13 18:20:11,033 - SL - INFO - 2769 - "/work/email_handler.py:2367" - _handle() - 8ad6dbbf-9d32-437e-8c04-e1f2e79b370e - Finish mail_from <>, rcpt_tos ['sl.lmycyibtheycyibsgm4deobwgboq.bzqmzgc5lk4ty@sl.local'], takes 0.004096031188964844 seconds with return code '550 SL E510 so such user'<<===
>>> handle_DATA() : '550 SL E510 so such user'

OBS_B_DONE
```

### K.4 Script C — canonical security boundary (GOOD/CORRUPTED/FUTURE/OLD signed)

> **Mock disclosure (read first):** this is the **only** observation script that imports `unittest.mock.patch`. The patch target is **`app.email_utils.time`** — the clock the *signer* `generate_verp_email()` reads — and it is applied **solely inside the helper `gen_shift()` while generating the two timestamp-boundary input addresses** (`FUTURE +6d` and `OLD -6d`), so that the software itself timestamps them beyond / within the 5-day `VERP_MESSAGE_LIFETIME`. The patch is a context manager scoped to address *generation* only; it patches **nothing** in the verifier `get_verp_info_from_email()`, the routing `handle()`, `_handle()`, `handle_DATA()`, the database, or the SMTP response path. Every **observed** value in this script (the `get_verp_info_from_email()` verdict and the canonical `handle()`/`handle_DATA()` status for each of the four addresses) is produced by the **real, unpatched** code. The `GOOD` and `CORRUPTED` addresses use no shim at all. This is the precise sense in which Phase A.2 states that no observed value was produced under a mock.

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_C_boundary.py
```

**Observation-script source (`obs_C_boundary.py`):**

```python
"""OBSERVATION SCRIPT C — Objective 4 security boundary of the SIGNED format.
Crafts, via the software's OWN signer generate_verp_email() (canonical signer) with a
time shim, four signed addresses: GOOD, CORRUPTED-signature, FUTURE-dated(+6d, beyond the
5-day lifetime), OLD-dated(-6d). Shows get_verp_info_from_email() verdict (NON-CANONICAL helper)
and the CANONICAL handle()/handle_DATA() outcome for each, on a VALID forward email_log_id."""
import os, asyncio
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from unittest.mock import patch
from aiosmtpd.smtp import Envelope
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import time as _t
import email_handler
import app.email_utils as eu
from app import config
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email, get_verp_info_from_email
from tests.utils import create_new_user

NONBOUNCE = ("From: attacker@evil.example\r\nTo: probe@sl.local\r\nSubject: hi\r\n"
             "Content-Type: text/plain\r\n\r\nhello\r\n")

def gen_shift(vt, oid, shift_seconds):
    base = _t.time()
    with patch("app.email_utils.time") as mt:
        mt.time.return_value = base + shift_seconds
        return generate_verp_email(vt, oid)

def run_handle(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    try: return "RETURNED %r" % (email_handler.handle(env, email.message_from_string(raw)),)
    except Exception as e: return "RAISED %s(%s)" % (type(e).__name__, e)

def run_handle_data(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    env.original_content = raw.encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

transaction = connection.begin()
with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    try:
        user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
        contact = Contact.create(user_id=user.id, alias_id=alias.id,
            website_email="contact@example.com", reply_email="rep@sl.local", commit=True)
        el = EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=False, commit=True)
        VID = el.id
        print("FIXTURE valid_forward_email_log_id=%s  VERP_MESSAGE_LIFETIME=%s days" % (VID, config.VERP_MESSAGE_LIFETIME/86400))

        good = generate_verp_email(VerpType.bounce_forward, VID)
        _lp, _dom = good.split("@"); _f = _lp.split(".")
        _sig = list(_f[2]); _sig[0] = ("b" if _sig[0] != "b" else "c")  # flip 1 char INSIDE the base32 signature
        corrupted = "%s.%s.%s@%s" % (_f[0], _f[1], "".join(_sig), _dom)
        future = gen_shift(VerpType.bounce_forward, VID, +6*86400)               # +6 days -> beyond 5d lifetime
        old = gen_shift(VerpType.bounce_forward, VID, -6*86400)                  # -6 days in the past

        for label, addr in [("GOOD (just signed)", good),
                            ("CORRUPTED signature", corrupted),
                            ("FUTURE-dated +6d (beyond 5d lifetime)", future),
                            ("OLD-dated -6d (in the past)", old)]:
            print("\n########## SIGNED %s ##########" % label)
            print("address = %r" % addr)
            print(">>> get_verp_info_from_email() [NON-CANONICAL helper] = %r" % (get_verp_info_from_email(addr),))
            print(">>> handle()      [CANONICAL, non-bounce probe] : %s" % run_handle("attacker@evil.example", addr, NONBOUNCE))
            print(">>> handle_DATA() [CANONICAL] : %r" % (run_handle_data("attacker@evil.example", addr, NONBOUNCE),))
    finally:
        transaction.rollback(); Session.rollback(); Session.close()
print("\nOBS_C_DONE")
```

**Complete verbatim transcript (`obs_C.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/xzfdnaldwuadqiuriwha
Upload files to local dir
>>> init logging <<<
2026-07-13 18:21:03,720 - SL - DEBUG - 2843 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-13 18:21:04,822 - SL - DEBUG - 2843 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 18:21:04,823 - SL - DEBUG - 2843 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 18:21:04,824 - SL - DEBUG - 2843 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:21:04,825 - SL - DEBUG - 2843 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:21:05,125 - SL - INFO - 2843 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:21:05,140 - SL - DEBUG - 2843 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email messed_corded584@sl.local
2026-07-13 18:21:05,148 - SL - INFO - 2843 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURE valid_forward_email_log_id=392  VERP_MESSAGE_LIFETIME=5.0 days

########## SIGNED GOOD (just signed) ##########
address = 'sl.lmycyibthezcyibsgm4deobwgfoq.a5s77tdbqy7i6@sl.local'
>>> get_verp_info_from_email() [NON-CANONICAL helper] = (<VerpType.bounce_forward: 0>, 392)
2026-07-13 18:21:05,166 - SL - INFO - 2843 - "/work/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:21:05,166 - SL - DEBUG - 2843 - "/work/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-13 18:21:05,167 - SL - DEBUG - 2843 - "/work/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm4deobwgfoq.a5s77tdbqy7i6@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
>>> handle()      [CANONICAL, non-bounce probe] : RAISED VERPForward(VERPForward )
2026-07-13 18:21:05,171 - SL - DEBUG - 2843 - "/work/app/log.py:24" - set_message_id() -  - set message_id 85a41f94-19fe-46cc-afb1-b6c0723a0001
2026-07-13 18:21:05,171 - SL - DEBUG - 2843 - "/work/email_handler.py:2342" - _handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:21:05,171 - SL - INFO - 2843 - "/work/email_handler.py:2343" - _handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibthezcyibsgm4deobwgfoq.a5s77tdbqy7i6@sl.local']
2026-07-13 18:21:05,171 - SL - INFO - 2843 - "/work/email_handler.py:1956" - handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:21:05,172 - SL - DEBUG - 2843 - "/work/email_handler.py:1963" - handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - Cannot parse Postfix queue ID from None None
2026-07-13 18:21:05,172 - SL - DEBUG - 2843 - "/work/email_handler.py:1980" - handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm4deobwgfoq.a5s77tdbqy7i6@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:21:05,174 - SL - WARNING - 2843 - "/work/email_handler.py:2309" - handle_DATA() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm4deobwgfoq.a5s77tdbqy7i6@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
>>> handle_DATA() [CANONICAL] : '250 SL E213 Unknown email ignored'

########## SIGNED CORRUPTED signature ##########
address = 'sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local'
>>> get_verp_info_from_email() [NON-CANONICAL helper] = None
2026-07-13 18:21:05,175 - SL - INFO - 2843 - "/work/email_handler.py:1956" - handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:21:05,175 - SL - DEBUG - 2843 - "/work/email_handler.py:1963" - handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - Cannot parse Postfix queue ID from None None
2026-07-13 18:21:05,176 - SL - DEBUG - 2843 - "/work/email_handler.py:1980" - handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:21:05,179 - SL - DEBUG - 2843 - "/work/email_handler.py:2202" - handle() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - Forward phase attacker@evil.example(attacker@evil.example) -> sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local
2026-07-13 18:21:05,186 - SL - DEBUG - 2843 - "/work/email_handler.py:545" - handle_forward() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - alias sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 18:21:05,192 - SL - INFO - 2843 - "/work/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - Cannot auto-create custom domain alias for sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local because there's no custom domain for sl.local
2026-07-13 18:21:05,192 - SL - INFO - 2843 - "/work/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - Cannot auto-create sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local since it has no directory separator
2026-07-13 18:21:05,192 - SL - DEBUG - 2843 - "/work/email_handler.py:551" - handle_forward() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - alias sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local cannot be created on-the-fly, return 550
>>> handle()      [CANONICAL, non-bounce probe] : RETURNED '550 SL E515 Email not exist'
2026-07-13 18:21:05,194 - SL - DEBUG - 2843 - "/work/app/log.py:24" - set_message_id() - 85a41f94-19fe-46cc-afb1-b6c0723a0001 - set message_id bb4597f0-64be-45e7-a0af-98ec8d6757bb
2026-07-13 18:21:05,194 - SL - DEBUG - 2843 - "/work/email_handler.py:2342" - _handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - ====>=====>====>====>====>====>====>====>
2026-07-13 18:21:05,194 - SL - INFO - 2843 - "/work/email_handler.py:2343" - _handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local']
2026-07-13 18:21:05,195 - SL - INFO - 2843 - "/work/email_handler.py:1956" - handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:21:05,195 - SL - DEBUG - 2843 - "/work/email_handler.py:1963" - handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Cannot parse Postfix queue ID from None None
2026-07-13 18:21:05,196 - SL - DEBUG - 2843 - "/work/email_handler.py:1980" - handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:21:05,199 - SL - DEBUG - 2843 - "/work/email_handler.py:2202" - handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Forward phase attacker@evil.example(attacker@evil.example) -> sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local
2026-07-13 18:21:05,205 - SL - DEBUG - 2843 - "/work/email_handler.py:545" - handle_forward() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - alias sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 18:21:05,210 - SL - INFO - 2843 - "/work/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Cannot auto-create custom domain alias for sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local because there's no custom domain for sl.local
2026-07-13 18:21:05,210 - SL - INFO - 2843 - "/work/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Cannot auto-create sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local since it has no directory separator
2026-07-13 18:21:05,210 - SL - DEBUG - 2843 - "/work/email_handler.py:551" - handle_forward() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - alias sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local cannot be created on-the-fly, return 550
2026-07-13 18:21:05,210 - SL - INFO - 2843 - "/work/email_handler.py:2367" - _handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibthezcyibsgm4deobwgfoq.b5s77tdbqy7i6@sl.local'], takes 0.016385793685913086 seconds with return code '550 SL E515 Email not exist'<<===
>>> handle_DATA() [CANONICAL] : '550 SL E515 Email not exist'

########## SIGNED FUTURE-dated +6d (beyond 5d lifetime) ##########
address = 'sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local'
>>> get_verp_info_from_email() [NON-CANONICAL helper] = None
2026-07-13 18:21:05,211 - SL - INFO - 2843 - "/work/email_handler.py:1956" - handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:21:05,211 - SL - DEBUG - 2843 - "/work/email_handler.py:1963" - handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Cannot parse Postfix queue ID from None None
2026-07-13 18:21:05,212 - SL - DEBUG - 2843 - "/work/email_handler.py:1980" - handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:21:05,216 - SL - DEBUG - 2843 - "/work/email_handler.py:2202" - handle() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Forward phase attacker@evil.example(attacker@evil.example) -> sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local
2026-07-13 18:21:05,222 - SL - DEBUG - 2843 - "/work/email_handler.py:545" - handle_forward() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - alias sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 18:21:05,227 - SL - INFO - 2843 - "/work/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Cannot auto-create custom domain alias for sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local because there's no custom domain for sl.local
2026-07-13 18:21:05,227 - SL - INFO - 2843 - "/work/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - Cannot auto-create sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local since it has no directory separator
2026-07-13 18:21:05,227 - SL - DEBUG - 2843 - "/work/email_handler.py:551" - handle_forward() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - alias sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local cannot be created on-the-fly, return 550
>>> handle()      [CANONICAL, non-bounce probe] : RETURNED '550 SL E515 Email not exist'
2026-07-13 18:21:05,229 - SL - DEBUG - 2843 - "/work/app/log.py:24" - set_message_id() - bb4597f0-64be-45e7-a0af-98ec8d6757bb - set message_id 7679c4ef-987b-4a2e-a76f-834d2e350eb5
2026-07-13 18:21:05,229 - SL - DEBUG - 2843 - "/work/email_handler.py:2342" - _handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:21:05,229 - SL - INFO - 2843 - "/work/email_handler.py:2343" - _handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local']
2026-07-13 18:21:05,229 - SL - INFO - 2843 - "/work/email_handler.py:1956" - handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:21:05,230 - SL - DEBUG - 2843 - "/work/email_handler.py:1963" - handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - Cannot parse Postfix queue ID from None None
2026-07-13 18:21:05,230 - SL - DEBUG - 2843 - "/work/email_handler.py:1980" - handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:21:05,234 - SL - DEBUG - 2843 - "/work/email_handler.py:2202" - handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - Forward phase attacker@evil.example(attacker@evil.example) -> sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local
2026-07-13 18:21:05,241 - SL - DEBUG - 2843 - "/work/email_handler.py:545" - handle_forward() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - alias sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 18:21:05,245 - SL - INFO - 2843 - "/work/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - Cannot auto-create custom domain alias for sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local because there's no custom domain for sl.local
2026-07-13 18:21:05,245 - SL - INFO - 2843 - "/work/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - Cannot auto-create sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local since it has no directory separator
2026-07-13 18:21:05,245 - SL - DEBUG - 2843 - "/work/email_handler.py:551" - handle_forward() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - alias sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local cannot be created on-the-fly, return 550
2026-07-13 18:21:05,246 - SL - INFO - 2843 - "/work/email_handler.py:2367" - _handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibthezcyibsgm4tcnjqgfoq.ycdkoap3lgzdq@sl.local'], takes 0.01710057258605957 seconds with return code '550 SL E515 Email not exist'<<===
>>> handle_DATA() [CANONICAL] : '550 SL E515 Email not exist'

########## SIGNED OLD-dated -6d (in the past) ##########
address = 'sl.lmycyibthezcyibsgm3timrsgfoq.bdfrd5kda2hog@sl.local'
>>> get_verp_info_from_email() [NON-CANONICAL helper] = (<VerpType.bounce_forward: 0>, 392)
2026-07-13 18:21:05,247 - SL - INFO - 2843 - "/work/email_handler.py:1956" - handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:21:05,247 - SL - DEBUG - 2843 - "/work/email_handler.py:1963" - handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - Cannot parse Postfix queue ID from None None
2026-07-13 18:21:05,248 - SL - DEBUG - 2843 - "/work/email_handler.py:1980" - handle() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm3timrsgfoq.bdfrd5kda2hog@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
>>> handle()      [CANONICAL, non-bounce probe] : RAISED VERPForward(VERPForward )
2026-07-13 18:21:05,253 - SL - DEBUG - 2843 - "/work/app/log.py:24" - set_message_id() - 7679c4ef-987b-4a2e-a76f-834d2e350eb5 - set message_id 99c60243-6f04-49cc-ac36-5512c3fea0a2
2026-07-13 18:21:05,253 - SL - DEBUG - 2843 - "/work/email_handler.py:2342" - _handle() - 99c60243-6f04-49cc-ac36-5512c3fea0a2 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:21:05,253 - SL - INFO - 2843 - "/work/email_handler.py:2343" - _handle() - 99c60243-6f04-49cc-ac36-5512c3fea0a2 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibthezcyibsgm3timrsgfoq.bdfrd5kda2hog@sl.local']
2026-07-13 18:21:05,254 - SL - INFO - 2843 - "/work/email_handler.py:1956" - handle() - 99c60243-6f04-49cc-ac36-5512c3fea0a2 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:21:05,254 - SL - DEBUG - 2843 - "/work/email_handler.py:1963" - handle() - 99c60243-6f04-49cc-ac36-5512c3fea0a2 - Cannot parse Postfix queue ID from None None
2026-07-13 18:21:05,255 - SL - DEBUG - 2843 - "/work/email_handler.py:1980" - handle() - 99c60243-6f04-49cc-ac36-5512c3fea0a2 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm3timrsgfoq.bdfrd5kda2hog@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:21:05,258 - SL - WARNING - 2843 - "/work/email_handler.py:2309" - handle_DATA() - 99c60243-6f04-49cc-ac36-5512c3fea0a2 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibthezcyibsgm3timrsgfoq.bdfrd5kda2hog@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
>>> handle_DATA() [CANONICAL] : '250 SL E213 Unknown email ignored'

OBS_C_DONE
```

### K.5 Script D — SPF suppression split rows (via _handle())

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_D_spf.py
```

**Observation-script source (`obs_D_spf.py`):**

```python
"""OBSERVATION SCRIPT D — Objective (SPF edge): the _handle() 5XX->E216 rewrite, split into
DISTINCT rows per SPF verdict {R_SPF_FAIL, R_SPF_SOFTFAIL, R_SPF_ALLOW, absent}. Uses the
project's own SPF EML templates via tests.utils.load_eml_file (canonical harness). Also shows
the inactive-user E510 (550) being silenced under SPF fail. All via MailHandler()._handle()."""
import os, arrow
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from aiosmtpd.smtp import Envelope
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email
from tests.utils import create_new_user, load_eml_file

def handle_spf(mail_from, rcpt, msg):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    return email_handler.MailHandler()._handle(env, msg)

def bounce_with_spf(spf_token):
    hdr = ""
    if spf_token:
        hdr = ("X-Spamd-Result: default: False [0.50 / 13.00];\n"
               " %s(0.00)[];\n ARC_NA(0.00)[]\n" % spf_token)
    return email.message_from_string(hdr +
        "From: MAILER-DAEMON@evil.example\nTo: probe@sl.local\nSubject: bounce\n"
        'Content-Type: multipart/report; report-type=delivery-status; boundary="b"\n\n'
        "--b\nContent-Type: text/plain\n\nDelivery failed\n"
        "--b\nContent-Type: message/delivery-status\n\nStatus: 5.1.1\n--b--\n")

transaction = connection.begin()
with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    try:
        user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
        INV = 99999999999999
        old = "%s%s%s" % (config.BOUNCE_PREFIX, INV, config.BOUNCE_SUFFIX)
        signed = generate_verp_email(VerpType.bounce_forward, INV)
        print("FIXTURE alias=%s INVALID_id=%s  old=%r  signed=%r" % (alias.email, INV, old, signed))

        def spf_msg(tok):
            if tok is None:
                return load_eml_file("no_spamd_header.eml", {"alias_email": alias.email})
            return load_eml_file("5xx_overwrite_spf.eml", {"alias_email": alias.email, "spf_result": tok})

        print("\n===== PART 1: INVALID id, OLD unsigned (bounce+INV+@sl.local), via _handle() =====")
        for tok in ["R_SPF_FAIL", "R_SPF_SOFTFAIL", "R_SPF_ALLOW", None]:
            m = spf_msg(tok)
            r = handle_spf(m["from"], old, m)
            print("SPF=%-14s -> _handle() = %r" % (str(tok), r))

        print("\n===== PART 2: INVALID id, NEW signed (generate_verp_email), via _handle() =====")
        for tok in ["R_SPF_FAIL", "R_SPF_SOFTFAIL", "R_SPF_ALLOW", None]:
            m = spf_msg(tok)
            r = handle_spf(m["from"], signed, m)
            print("SPF=%-14s -> _handle() = %r" % (str(tok), r))

        print("\n===== PART 3: INACTIVE user E510 (valid id, bounce msg + rspamd hdr), via _handle() =====")
        u2 = create_new_user(); a2 = Alias.create_new_random(u2); Session.commit()
        c2 = Contact.create(user_id=u2.id, alias_id=a2.id, website_email="c2@example.com", reply_email="rep2@sl.local", commit=True)
        el2 = EmailLog.create(user_id=u2.id, contact_id=c2.id, alias_id=a2.id, is_reply=False, commit=True)
        u2.delete_on = arrow.now().shift(days=1); Session.commit()
        rcpt = generate_verp_email(VerpType.bounce_forward, el2.id)
        print("u2.is_active()=%s email_log_id=%s (bounce->E510 without SPF suppression)" % (u2.is_active(), el2.id))
        for tok in ["R_SPF_FAIL", "R_SPF_ALLOW", None]:
            r = handle_spf("<>", rcpt, bounce_with_spf(tok))
            print("SPF=%-14s -> _handle() = %r" % (str(tok), r))
    finally:
        transaction.rollback(); Session.rollback(); Session.close()
print("\nOBS_D_DONE")
```

**Complete verbatim transcript (`obs_D.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/yomjymtwiggiavvyzbxa
Upload files to local dir
>>> init logging <<<
2026-07-13 18:23:01,070 - SL - DEBUG - 2890 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-13 18:23:02,265 - SL - DEBUG - 2890 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 18:23:02,267 - SL - DEBUG - 2890 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 18:23:02,267 - SL - DEBUG - 2890 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:23:02,268 - SL - DEBUG - 2890 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:23:02,585 - SL - INFO - 2890 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:23:02,601 - SL - DEBUG - 2890 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email sprout_seethe206@sl.local
2026-07-13 18:23:02,611 - SL - INFO - 2890 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURE alias=sprout_seethe206@sl.local INVALID_id=99999999999999  old='bounce+99999999999999+@sl.local'  signed='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'

===== PART 1: INVALID id, OLD unsigned (bounce+INV+@sl.local), via _handle() =====
2026-07-13 18:23:02,619 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() -  - set message_id ab03a9c1-0917-4c19-bcd7-5ef5678edd97
2026-07-13 18:23:02,620 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - ab03a9c1-0917-4c19-bcd7-5ef5678edd97 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,620 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - ab03a9c1-0917-4c19-bcd7-5ef5678edd97 - New message, mail from somewhere@rainbow.com, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:23:02,621 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - ab03a9c1-0917-4c19-bcd7-5ef5678edd97 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,621 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - ab03a9c1-0917-4c19-bcd7-5ef5678edd97 - set message_id 6D8C13F069
2026-07-13 18:23:02,622 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:somewhere@rainbow.com, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:somewhere@rainbow.com, header_to:sprout_seethe206@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'sprout_seethe206@sl.local'), ('From', 'somewhere@rainbow.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('X-Spamd-Result', 'default: False [0.50 / 13.00];\n        MID_RHS_NOT_FQDN(0.50)[];\n        DMARC_NA(0.10);\n        MIME_GOOD(-0.10)[text/plain];\n        MIME_TRACE(0.00)[0:+];\n        TO_DN_NONE(0.00)[];\n        R_SPF_FAIL(0.00[];\n        TO_MATCH_ENVRCPT_ALL(0.00)[];\n        ARC_NA(0.00)[]'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,629 - SL - WARNING - 2890 - "/work/email_handler.py:2066" - handle() - 6D8C13F069 - No such email log
2026-07-13 18:23:02,629 - SL - INFO - 2890 - "/work/email_handler.py:2362" - _handle() - 6D8C13F069 - Replacing 5XX to 216 status because the return-path failed the spf check
2026-07-13 18:23:02,629 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6D8C13F069 - Finish mail_from somewhere@rainbow.com, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.009947538375854492 seconds with return code '250 SL E216 Handled spf policy'<<===
SPF=R_SPF_FAIL     -> _handle() = '250 SL E216 Handled spf policy'
2026-07-13 18:23:02,631 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6D8C13F069 - set message_id 9b4e3e58-933e-46d0-9d1d-28e9e1b09eba
2026-07-13 18:23:02,631 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - 9b4e3e58-933e-46d0-9d1d-28e9e1b09eba - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,631 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - 9b4e3e58-933e-46d0-9d1d-28e9e1b09eba - New message, mail from somewhere@rainbow.com, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:23:02,632 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - 9b4e3e58-933e-46d0-9d1d-28e9e1b09eba - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,632 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 9b4e3e58-933e-46d0-9d1d-28e9e1b09eba - set message_id 6D8C13F069
2026-07-13 18:23:02,633 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:somewhere@rainbow.com, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:somewhere@rainbow.com, header_to:sprout_seethe206@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'sprout_seethe206@sl.local'), ('From', 'somewhere@rainbow.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('X-Spamd-Result', 'default: False [0.50 / 13.00];\n        MID_RHS_NOT_FQDN(0.50)[];\n        DMARC_NA(0.10);\n        MIME_GOOD(-0.10)[text/plain];\n        MIME_TRACE(0.00)[0:+];\n        TO_DN_NONE(0.00)[];\n        R_SPF_SOFTFAIL(0.00[];\n        TO_MATCH_ENVRCPT_ALL(0.00)[];\n        ARC_NA(0.00)[]'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,638 - SL - WARNING - 2890 - "/work/email_handler.py:2066" - handle() - 6D8C13F069 - No such email log
2026-07-13 18:23:02,638 - SL - INFO - 2890 - "/work/email_handler.py:2362" - _handle() - 6D8C13F069 - Replacing 5XX to 216 status because the return-path failed the spf check
2026-07-13 18:23:02,638 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6D8C13F069 - Finish mail_from somewhere@rainbow.com, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.006681919097900391 seconds with return code '250 SL E216 Handled spf policy'<<===
SPF=R_SPF_SOFTFAIL -> _handle() = '250 SL E216 Handled spf policy'
2026-07-13 18:23:02,640 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6D8C13F069 - set message_id 76ce5384-d566-4522-9b6c-74e91f9bfa34
2026-07-13 18:23:02,640 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - 76ce5384-d566-4522-9b6c-74e91f9bfa34 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,640 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - 76ce5384-d566-4522-9b6c-74e91f9bfa34 - New message, mail from somewhere@rainbow.com, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:23:02,641 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - 76ce5384-d566-4522-9b6c-74e91f9bfa34 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,641 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 76ce5384-d566-4522-9b6c-74e91f9bfa34 - set message_id 6D8C13F069
2026-07-13 18:23:02,642 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:somewhere@rainbow.com, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:somewhere@rainbow.com, header_to:sprout_seethe206@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'sprout_seethe206@sl.local'), ('From', 'somewhere@rainbow.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('X-Spamd-Result', 'default: False [0.50 / 13.00];\n        MID_RHS_NOT_FQDN(0.50)[];\n        DMARC_NA(0.10);\n        MIME_GOOD(-0.10)[text/plain];\n        MIME_TRACE(0.00)[0:+];\n        TO_DN_NONE(0.00)[];\n        R_SPF_ALLOW(0.00[];\n        TO_MATCH_ENVRCPT_ALL(0.00)[];\n        ARC_NA(0.00)[]'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,647 - SL - WARNING - 2890 - "/work/email_handler.py:2066" - handle() - 6D8C13F069 - No such email log
2026-07-13 18:23:02,647 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6D8C13F069 - Finish mail_from somewhere@rainbow.com, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.0071680545806884766 seconds with return code '550 SL E512 No such email log'<<===
SPF=R_SPF_ALLOW    -> _handle() = '550 SL E512 No such email log'
2026-07-13 18:23:02,649 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6D8C13F069 - set message_id ed17917a-3b5b-4f69-96dc-b47b277ed9a5
2026-07-13 18:23:02,649 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - ed17917a-3b5b-4f69-96dc-b47b277ed9a5 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,649 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - ed17917a-3b5b-4f69-96dc-b47b277ed9a5 - New message, mail from somewhere@rainbow.com, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:23:02,650 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - ed17917a-3b5b-4f69-96dc-b47b277ed9a5 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,650 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - ed17917a-3b5b-4f69-96dc-b47b277ed9a5 - set message_id 6D8C13F069
2026-07-13 18:23:02,651 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:somewhere@rainbow.com, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:somewhere@rainbow.com, header_to:sprout_seethe206@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'sprout_seethe206@sl.local'), ('From', 'somewhere@rainbow.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,656 - SL - WARNING - 2890 - "/work/email_handler.py:2066" - handle() - 6D8C13F069 - No such email log
2026-07-13 18:23:02,656 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6D8C13F069 - Finish mail_from somewhere@rainbow.com, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.007248878479003906 seconds with return code '550 SL E512 No such email log'<<===
SPF=None           -> _handle() = '550 SL E512 No such email log'

===== PART 2: INVALID id, NEW signed (generate_verp_email), via _handle() =====
2026-07-13 18:23:02,658 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6D8C13F069 - set message_id 5b85d270-8938-41ca-a50b-9a3e4f99eaca
2026-07-13 18:23:02,658 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - 5b85d270-8938-41ca-a50b-9a3e4f99eaca - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,658 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - 5b85d270-8938-41ca-a50b-9a3e4f99eaca - New message, mail from somewhere@rainbow.com, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local']
2026-07-13 18:23:02,659 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - 5b85d270-8938-41ca-a50b-9a3e4f99eaca - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,660 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 5b85d270-8938-41ca-a50b-9a3e4f99eaca - set message_id 6D8C13F069
2026-07-13 18:23:02,661 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:somewhere@rainbow.com, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'], header_from:somewhere@rainbow.com, header_to:sprout_seethe206@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'sprout_seethe206@sl.local'), ('From', 'somewhere@rainbow.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('X-Spamd-Result', 'default: False [0.50 / 13.00];\n        MID_RHS_NOT_FQDN(0.50)[];\n        DMARC_NA(0.10);\n        MIME_GOOD(-0.10)[text/plain];\n        MIME_TRACE(0.00)[0:+];\n        TO_DN_NONE(0.00)[];\n        R_SPF_FAIL(0.00[];\n        TO_MATCH_ENVRCPT_ALL(0.00)[];\n        ARC_NA(0.00)[]'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,665 - SL - WARNING - 2890 - "/work/email_handler.py:2066" - handle() - 6D8C13F069 - No such email log
2026-07-13 18:23:02,665 - SL - INFO - 2890 - "/work/email_handler.py:2362" - _handle() - 6D8C13F069 - Replacing 5XX to 216 status because the return-path failed the spf check
2026-07-13 18:23:02,665 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6D8C13F069 - Finish mail_from somewhere@rainbow.com, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'], takes 0.007179737091064453 seconds with return code '250 SL E216 Handled spf policy'<<===
SPF=R_SPF_FAIL     -> _handle() = '250 SL E216 Handled spf policy'
2026-07-13 18:23:02,667 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6D8C13F069 - set message_id f7540a72-9e80-4145-934b-63038108675b
2026-07-13 18:23:02,667 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - f7540a72-9e80-4145-934b-63038108675b - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,667 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - f7540a72-9e80-4145-934b-63038108675b - New message, mail from somewhere@rainbow.com, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local']
2026-07-13 18:23:02,668 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - f7540a72-9e80-4145-934b-63038108675b - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,668 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - f7540a72-9e80-4145-934b-63038108675b - set message_id 6D8C13F069
2026-07-13 18:23:02,669 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:somewhere@rainbow.com, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'], header_from:somewhere@rainbow.com, header_to:sprout_seethe206@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'sprout_seethe206@sl.local'), ('From', 'somewhere@rainbow.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('X-Spamd-Result', 'default: False [0.50 / 13.00];\n        MID_RHS_NOT_FQDN(0.50)[];\n        DMARC_NA(0.10);\n        MIME_GOOD(-0.10)[text/plain];\n        MIME_TRACE(0.00)[0:+];\n        TO_DN_NONE(0.00)[];\n        R_SPF_SOFTFAIL(0.00[];\n        TO_MATCH_ENVRCPT_ALL(0.00)[];\n        ARC_NA(0.00)[]'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,674 - SL - WARNING - 2890 - "/work/email_handler.py:2066" - handle() - 6D8C13F069 - No such email log
2026-07-13 18:23:02,674 - SL - INFO - 2890 - "/work/email_handler.py:2362" - _handle() - 6D8C13F069 - Replacing 5XX to 216 status because the return-path failed the spf check
2026-07-13 18:23:02,674 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6D8C13F069 - Finish mail_from somewhere@rainbow.com, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'], takes 0.0066051483154296875 seconds with return code '250 SL E216 Handled spf policy'<<===
SPF=R_SPF_SOFTFAIL -> _handle() = '250 SL E216 Handled spf policy'
2026-07-13 18:23:02,675 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6D8C13F069 - set message_id ad5d013c-9d6d-4c16-a45b-60fb0cd0ec87
2026-07-13 18:23:02,675 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - ad5d013c-9d6d-4c16-a45b-60fb0cd0ec87 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,675 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - ad5d013c-9d6d-4c16-a45b-60fb0cd0ec87 - New message, mail from somewhere@rainbow.com, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local']
2026-07-13 18:23:02,676 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - ad5d013c-9d6d-4c16-a45b-60fb0cd0ec87 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,676 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - ad5d013c-9d6d-4c16-a45b-60fb0cd0ec87 - set message_id 6D8C13F069
2026-07-13 18:23:02,677 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:somewhere@rainbow.com, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'], header_from:somewhere@rainbow.com, header_to:sprout_seethe206@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'sprout_seethe206@sl.local'), ('From', 'somewhere@rainbow.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('X-Spamd-Result', 'default: False [0.50 / 13.00];\n        MID_RHS_NOT_FQDN(0.50)[];\n        DMARC_NA(0.10);\n        MIME_GOOD(-0.10)[text/plain];\n        MIME_TRACE(0.00)[0:+];\n        TO_DN_NONE(0.00)[];\n        R_SPF_ALLOW(0.00[];\n        TO_MATCH_ENVRCPT_ALL(0.00)[];\n        ARC_NA(0.00)[]'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,681 - SL - WARNING - 2890 - "/work/email_handler.py:2066" - handle() - 6D8C13F069 - No such email log
2026-07-13 18:23:02,681 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6D8C13F069 - Finish mail_from somewhere@rainbow.com, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'], takes 0.005556344985961914 seconds with return code '550 SL E512 No such email log'<<===
SPF=R_SPF_ALLOW    -> _handle() = '550 SL E512 No such email log'
2026-07-13 18:23:02,682 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6D8C13F069 - set message_id ad378a5d-fbb0-45ee-971c-cff886f62847
2026-07-13 18:23:02,682 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - ad378a5d-fbb0-45ee-971c-cff886f62847 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,682 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - ad378a5d-fbb0-45ee-971c-cff886f62847 - New message, mail from somewhere@rainbow.com, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local']
2026-07-13 18:23:02,683 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - ad378a5d-fbb0-45ee-971c-cff886f62847 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,683 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - ad378a5d-fbb0-45ee-971c-cff886f62847 - set message_id 6D8C13F069
2026-07-13 18:23:02,684 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:somewhere@rainbow.com, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'], header_from:somewhere@rainbow.com, header_to:sprout_seethe206@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'sprout_seethe206@sl.local'), ('From', 'somewhere@rainbow.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,687 - SL - WARNING - 2890 - "/work/email_handler.py:2066" - handle() - 6D8C13F069 - No such email log
2026-07-13 18:23:02,688 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6D8C13F069 - Finish mail_from somewhere@rainbow.com, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygyzv2.dxsyw4ucgat5y@sl.local'], takes 0.00557255744934082 seconds with return code '550 SL E512 No such email log'<<===
SPF=None           -> _handle() = '550 SL E512 No such email log'

===== PART 3: INACTIVE user E510 (valid id, bounce msg + rspamd hdr), via _handle() =====
2026-07-13 18:23:02,950 - SL - INFO - 2890 - "/work/app/events/event_dispatcher.py:62" - send_event() - 6D8C13F069 - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:23:02,963 - SL - DEBUG - 2890 - "/work/app/models.py:1459" - generate_random_alias_email() - 6D8C13F069 - generate email ceased_sloped484@sl.local
2026-07-13 18:23:02,972 - SL - INFO - 2890 - "/work/app/events/event_dispatcher.py:62" - send_event() - 6D8C13F069 - Not sending events because webhook is not configured and allowed to be empty
u2.is_active()=False email_log_id=393 (bounce->E510 without SPF suppression)
2026-07-13 18:23:02,993 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6D8C13F069 - set message_id 5fb1113d-372a-4afc-a4e2-752e9f027764
2026-07-13 18:23:02,993 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,993 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - New message, mail from <>, rctp tos ['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local']
2026-07-13 18:23:02,994 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:02,994 - SL - DEBUG - 2890 - "/work/email_handler.py:1963" - handle() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:02,995 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('X-Spamd-Result', 'default: False [0.50 / 13.00];\n R_SPF_FAIL(0.00)[];\n ARC_NA(0.00)[]'), ('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:02,998 - SL - DEBUG - 2890 - "/work/email_handler.py:1862" - handle_bounce() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - handle bounce for <EmailLog 393>, phase=forward, contact=<Contact 159 c2@example.com 832>, alias=<Alias 832 ceased_sloped484@sl.local>
2026-07-13 18:23:02,998 - SL - DEBUG - 2890 - "/work/email_handler.py:1870" - handle_bounce() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - User <User 484 Test User user_ud5n2qn2iu@mailbox.test> is not active
2026-07-13 18:23:02,998 - SL - INFO - 2890 - "/work/email_handler.py:2362" - _handle() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - Replacing 5XX to 216 status because the return-path failed the spf check
2026-07-13 18:23:02,998 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - Finish mail_from <>, rcpt_tos ['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local'], takes 0.005606174468994141 seconds with return code '250 SL E216 Handled spf policy'<<===
SPF=R_SPF_FAIL     -> _handle() = '250 SL E216 Handled spf policy'
2026-07-13 18:23:02,999 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 5fb1113d-372a-4afc-a4e2-752e9f027764 - set message_id 6ac86331-d871-486a-87d5-441c4f524014
2026-07-13 18:23:02,999 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - 6ac86331-d871-486a-87d5-441c4f524014 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:02,999 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - 6ac86331-d871-486a-87d5-441c4f524014 - New message, mail from <>, rctp tos ['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local']
2026-07-13 18:23:03,000 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - 6ac86331-d871-486a-87d5-441c4f524014 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:03,000 - SL - DEBUG - 2890 - "/work/email_handler.py:1963" - handle() - 6ac86331-d871-486a-87d5-441c4f524014 - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:03,001 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 6ac86331-d871-486a-87d5-441c4f524014 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('X-Spamd-Result', 'default: False [0.50 / 13.00];\n R_SPF_ALLOW(0.00)[];\n ARC_NA(0.00)[]'), ('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:03,008 - SL - DEBUG - 2890 - "/work/email_handler.py:1862" - handle_bounce() - 6ac86331-d871-486a-87d5-441c4f524014 - handle bounce for <EmailLog 393>, phase=forward, contact=<Contact 159 c2@example.com 832>, alias=<Alias 832 ceased_sloped484@sl.local>
2026-07-13 18:23:03,011 - SL - DEBUG - 2890 - "/work/email_handler.py:1870" - handle_bounce() - 6ac86331-d871-486a-87d5-441c4f524014 - User <User 484 Test User user_ud5n2qn2iu@mailbox.test> is not active
2026-07-13 18:23:03,011 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 6ac86331-d871-486a-87d5-441c4f524014 - Finish mail_from <>, rcpt_tos ['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local'], takes 0.011799335479736328 seconds with return code '550 SL E510 so such user'<<===
SPF=R_SPF_ALLOW    -> _handle() = '550 SL E510 so such user'
2026-07-13 18:23:03,012 - SL - DEBUG - 2890 - "/work/app/log.py:24" - set_message_id() - 6ac86331-d871-486a-87d5-441c4f524014 - set message_id 8c18c15b-d875-47a3-bc9b-b6caebb484ee
2026-07-13 18:23:03,012 - SL - DEBUG - 2890 - "/work/email_handler.py:2342" - _handle() - 8c18c15b-d875-47a3-bc9b-b6caebb484ee - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:03,012 - SL - INFO - 2890 - "/work/email_handler.py:2343" - _handle() - 8c18c15b-d875-47a3-bc9b-b6caebb484ee - New message, mail from <>, rctp tos ['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local']
2026-07-13 18:23:03,012 - SL - INFO - 2890 - "/work/email_handler.py:1956" - handle() - 8c18c15b-d875-47a3-bc9b-b6caebb484ee - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:03,013 - SL - DEBUG - 2890 - "/work/email_handler.py:1963" - handle() - 8c18c15b-d875-47a3-bc9b-b6caebb484ee - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:03,013 - SL - DEBUG - 2890 - "/work/email_handler.py:1980" - handle() - 8c18c15b-d875-47a3-bc9b-b6caebb484ee - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:03,020 - SL - DEBUG - 2890 - "/work/email_handler.py:1862" - handle_bounce() - 8c18c15b-d875-47a3-bc9b-b6caebb484ee - handle bounce for <EmailLog 393>, phase=forward, contact=<Contact 159 c2@example.com 832>, alias=<Alias 832 ceased_sloped484@sl.local>
2026-07-13 18:23:03,021 - SL - DEBUG - 2890 - "/work/email_handler.py:1870" - handle_bounce() - 8c18c15b-d875-47a3-bc9b-b6caebb484ee - User <User 484 Test User user_ud5n2qn2iu@mailbox.test> is not active
2026-07-13 18:23:03,021 - SL - INFO - 2890 - "/work/email_handler.py:2367" - _handle() - 8c18c15b-d875-47a3-bc9b-b6caebb484ee - Finish mail_from <>, rcpt_tos ['sl.lmycyibthezsyibsgm4deobwgnoq.r5ax4uxqrezj4@sl.local'], takes 0.00928950309753418 seconds with return code '550 SL E510 so such user'<<===
SPF=None           -> _handle() = '550 SL E510 so such user'

OBS_D_DONE
```

### K.5b Script D2 — SPF asymmetry (via handle_DATA())

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_D2_spf_asym.py
```

**Observation-script source (`obs_D2_spf_asym.py`):**

```python
"""OBSERVATION SCRIPT D2 — SPF-rewrite ASYMMETRY via the full wire path handle_DATA().
Shows that a RETURNED 5XX (invalid id -> E512) is rewritten to E216 under SPF fail, but a
RAISED VERPForward (valid id, non-bounce -> E213) BYPASSES the _handle SPF rewrite (exception
leaves _handle before the rewrite code). Hence under SPF-fail the oracle survives at the STRING
level: invalid -> '250 SL E216 ...' vs valid-non-bounce -> '250 SL E213 ...'."""
import os, asyncio
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from aiosmtpd.smtp import Envelope
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email
from tests.utils import create_new_user

def nonbounce_with_spf(tok):
    hdr = ("X-Spamd-Result: default: False [0.50 / 13.00];\n %s(0.00)[];\n ARC_NA(0.00)[]\n" % tok) if tok else ""
    return (hdr + "From: attacker@evil.example\nTo: probe@sl.local\nSubject: hi\nContent-Type: text/plain\n\nhello\n")

def run_handle_data(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]; env.original_content = raw.encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

transaction = connection.begin()
with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    try:
        user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
        contact = Contact.create(user_id=user.id, alias_id=alias.id, website_email="c@example.com", reply_email="rep@sl.local", commit=True)
        el = EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=False, commit=True)
        VID = el.id; INV = 99999999999999
        print("FIXTURE valid_id=%s invalid_id=%s" % (VID, INV))
        for tok in ["R_SPF_FAIL", "R_SPF_ALLOW", None]:
            inv_addr = "%s%s%s" % (config.BOUNCE_PREFIX, INV, config.BOUNCE_SUFFIX)
            val_addr = "%s%s%s" % (config.BOUNCE_PREFIX, VID, config.BOUNCE_SUFFIX)
            r_inv = run_handle_data("attacker@evil.example", inv_addr, nonbounce_with_spf(tok))
            r_val = run_handle_data("attacker@evil.example", val_addr, nonbounce_with_spf(tok))
            print("SPF=%-12s | INVALID non-bounce handle_DATA()=%r | VALID non-bounce handle_DATA()=%r"
                  % (str(tok), r_inv, r_val))
    finally:
        transaction.rollback(); Session.rollback(); Session.close()
print("\nOBS_D2_DONE")
```

**Complete verbatim transcript (`obs_D2.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/zpmunuxuikirzaplouiz
Upload files to local dir
>>> init logging <<<
2026-07-13 18:23:58,243 - SL - DEBUG - 2950 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-13 18:23:59,371 - SL - DEBUG - 2950 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 18:23:59,373 - SL - DEBUG - 2950 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 18:23:59,374 - SL - DEBUG - 2950 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:23:59,374 - SL - DEBUG - 2950 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:23:59,691 - SL - INFO - 2950 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:23:59,706 - SL - DEBUG - 2950 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email sabres_spiked365@sl.local
2026-07-13 18:23:59,715 - SL - INFO - 2950 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURE valid_id=395 invalid_id=99999999999999
2026-07-13 18:23:59,735 - SL - DEBUG - 2950 - "/work/app/log.py:24" - set_message_id() -  - set message_id 3118e862-ec01-4d7f-8c67-29ec5596c1e1
2026-07-13 18:23:59,735 - SL - DEBUG - 2950 - "/work/email_handler.py:2342" - _handle() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:59,735 - SL - INFO - 2950 - "/work/email_handler.py:2343" - _handle() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:23:59,736 - SL - INFO - 2950 - "/work/email_handler.py:1956" - handle() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:59,736 - SL - DEBUG - 2950 - "/work/email_handler.py:1963" - handle() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:59,737 - SL - DEBUG - 2950 - "/work/email_handler.py:1980" - handle() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('X-Spamd-Result', 'default: False [0.50 / 13.00];\n R_SPF_FAIL(0.00)[];\n ARC_NA(0.00)[]'), ('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:59,742 - SL - WARNING - 2950 - "/work/email_handler.py:2066" - handle() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - No such email log
2026-07-13 18:23:59,742 - SL - INFO - 2950 - "/work/email_handler.py:2362" - _handle() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - Replacing 5XX to 216 status because the return-path failed the spf check
2026-07-13 18:23:59,742 - SL - INFO - 2950 - "/work/email_handler.py:2367" - _handle() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.006909608840942383 seconds with return code '250 SL E216 Handled spf policy'<<===
2026-07-13 18:23:59,743 - SL - DEBUG - 2950 - "/work/app/log.py:24" - set_message_id() - 3118e862-ec01-4d7f-8c67-29ec5596c1e1 - set message_id ce286b9e-0ec0-400b-98b3-ea256151013e
2026-07-13 18:23:59,743 - SL - DEBUG - 2950 - "/work/email_handler.py:2342" - _handle() - ce286b9e-0ec0-400b-98b3-ea256151013e - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:59,743 - SL - INFO - 2950 - "/work/email_handler.py:2343" - _handle() - ce286b9e-0ec0-400b-98b3-ea256151013e - New message, mail from attacker@evil.example, rctp tos ['bounce+395+@sl.local']
2026-07-13 18:23:59,743 - SL - INFO - 2950 - "/work/email_handler.py:1956" - handle() - ce286b9e-0ec0-400b-98b3-ea256151013e - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:59,744 - SL - DEBUG - 2950 - "/work/email_handler.py:1963" - handle() - ce286b9e-0ec0-400b-98b3-ea256151013e - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:59,744 - SL - DEBUG - 2950 - "/work/email_handler.py:1980" - handle() - ce286b9e-0ec0-400b-98b3-ea256151013e - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+395+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('X-Spamd-Result', 'default: False [0.50 / 13.00];\n R_SPF_FAIL(0.00)[];\n ARC_NA(0.00)[]'), ('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:59,748 - SL - WARNING - 2950 - "/work/email_handler.py:2309" - handle_DATA() - ce286b9e-0ec0-400b-98b3-ea256151013e - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+395+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
SPF=R_SPF_FAIL   | INVALID non-bounce handle_DATA()='250 SL E216 Handled spf policy' | VALID non-bounce handle_DATA()='250 SL E213 Unknown email ignored'
2026-07-13 18:23:59,748 - SL - DEBUG - 2950 - "/work/app/log.py:24" - set_message_id() - ce286b9e-0ec0-400b-98b3-ea256151013e - set message_id 1dfc84dc-c2cf-421c-8007-df28a6b07480
2026-07-13 18:23:59,748 - SL - DEBUG - 2950 - "/work/email_handler.py:2342" - _handle() - 1dfc84dc-c2cf-421c-8007-df28a6b07480 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:59,748 - SL - INFO - 2950 - "/work/email_handler.py:2343" - _handle() - 1dfc84dc-c2cf-421c-8007-df28a6b07480 - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:23:59,749 - SL - INFO - 2950 - "/work/email_handler.py:1956" - handle() - 1dfc84dc-c2cf-421c-8007-df28a6b07480 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:59,749 - SL - DEBUG - 2950 - "/work/email_handler.py:1963" - handle() - 1dfc84dc-c2cf-421c-8007-df28a6b07480 - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:59,750 - SL - DEBUG - 2950 - "/work/email_handler.py:1980" - handle() - 1dfc84dc-c2cf-421c-8007-df28a6b07480 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('X-Spamd-Result', 'default: False [0.50 / 13.00];\n R_SPF_ALLOW(0.00)[];\n ARC_NA(0.00)[]'), ('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:59,753 - SL - WARNING - 2950 - "/work/email_handler.py:2066" - handle() - 1dfc84dc-c2cf-421c-8007-df28a6b07480 - No such email log
2026-07-13 18:23:59,753 - SL - INFO - 2950 - "/work/email_handler.py:2367" - _handle() - 1dfc84dc-c2cf-421c-8007-df28a6b07480 - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.00466609001159668 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:23:59,754 - SL - DEBUG - 2950 - "/work/app/log.py:24" - set_message_id() - 1dfc84dc-c2cf-421c-8007-df28a6b07480 - set message_id 19ce9bda-7167-4aa6-86c0-8cef94309bb0
2026-07-13 18:23:59,754 - SL - DEBUG - 2950 - "/work/email_handler.py:2342" - _handle() - 19ce9bda-7167-4aa6-86c0-8cef94309bb0 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:59,754 - SL - INFO - 2950 - "/work/email_handler.py:2343" - _handle() - 19ce9bda-7167-4aa6-86c0-8cef94309bb0 - New message, mail from attacker@evil.example, rctp tos ['bounce+395+@sl.local']
2026-07-13 18:23:59,754 - SL - INFO - 2950 - "/work/email_handler.py:1956" - handle() - 19ce9bda-7167-4aa6-86c0-8cef94309bb0 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:59,754 - SL - DEBUG - 2950 - "/work/email_handler.py:1963" - handle() - 19ce9bda-7167-4aa6-86c0-8cef94309bb0 - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:59,755 - SL - DEBUG - 2950 - "/work/email_handler.py:1980" - handle() - 19ce9bda-7167-4aa6-86c0-8cef94309bb0 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+395+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('X-Spamd-Result', 'default: False [0.50 / 13.00];\n R_SPF_ALLOW(0.00)[];\n ARC_NA(0.00)[]'), ('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:59,761 - SL - WARNING - 2950 - "/work/email_handler.py:2309" - handle_DATA() - 19ce9bda-7167-4aa6-86c0-8cef94309bb0 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+395+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
SPF=R_SPF_ALLOW  | INVALID non-bounce handle_DATA()='550 SL E512 No such email log' | VALID non-bounce handle_DATA()='250 SL E213 Unknown email ignored'
2026-07-13 18:23:59,762 - SL - DEBUG - 2950 - "/work/app/log.py:24" - set_message_id() - 19ce9bda-7167-4aa6-86c0-8cef94309bb0 - set message_id f5ec1f76-9955-48ed-bc5b-cd55cfb382ee
2026-07-13 18:23:59,762 - SL - DEBUG - 2950 - "/work/email_handler.py:2342" - _handle() - f5ec1f76-9955-48ed-bc5b-cd55cfb382ee - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:59,762 - SL - INFO - 2950 - "/work/email_handler.py:2343" - _handle() - f5ec1f76-9955-48ed-bc5b-cd55cfb382ee - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:23:59,762 - SL - INFO - 2950 - "/work/email_handler.py:1956" - handle() - f5ec1f76-9955-48ed-bc5b-cd55cfb382ee - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:59,762 - SL - DEBUG - 2950 - "/work/email_handler.py:1963" - handle() - f5ec1f76-9955-48ed-bc5b-cd55cfb382ee - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:59,763 - SL - DEBUG - 2950 - "/work/email_handler.py:1980" - handle() - f5ec1f76-9955-48ed-bc5b-cd55cfb382ee - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:59,766 - SL - WARNING - 2950 - "/work/email_handler.py:2066" - handle() - f5ec1f76-9955-48ed-bc5b-cd55cfb382ee - No such email log
2026-07-13 18:23:59,767 - SL - INFO - 2950 - "/work/email_handler.py:2367" - _handle() - f5ec1f76-9955-48ed-bc5b-cd55cfb382ee - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.0050506591796875 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:23:59,767 - SL - DEBUG - 2950 - "/work/app/log.py:24" - set_message_id() - f5ec1f76-9955-48ed-bc5b-cd55cfb382ee - set message_id 2a33fd01-d099-4ab4-b456-b73a0cd51c44
2026-07-13 18:23:59,767 - SL - DEBUG - 2950 - "/work/email_handler.py:2342" - _handle() - 2a33fd01-d099-4ab4-b456-b73a0cd51c44 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:23:59,767 - SL - INFO - 2950 - "/work/email_handler.py:2343" - _handle() - 2a33fd01-d099-4ab4-b456-b73a0cd51c44 - New message, mail from attacker@evil.example, rctp tos ['bounce+395+@sl.local']
2026-07-13 18:23:59,768 - SL - INFO - 2950 - "/work/email_handler.py:1956" - handle() - 2a33fd01-d099-4ab4-b456-b73a0cd51c44 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:23:59,768 - SL - DEBUG - 2950 - "/work/email_handler.py:1963" - handle() - 2a33fd01-d099-4ab4-b456-b73a0cd51c44 - Cannot parse Postfix queue ID from None None
2026-07-13 18:23:59,769 - SL - DEBUG - 2950 - "/work/email_handler.py:1980" - handle() - 2a33fd01-d099-4ab4-b456-b73a0cd51c44 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+395+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:23:59,772 - SL - WARNING - 2950 - "/work/email_handler.py:2309" - handle_DATA() - 2a33fd01-d099-4ab4-b456-b73a0cd51c44 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+395+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
SPF=None         | INVALID non-bounce handle_DATA()='550 SL E512 No such email log' | VALID non-bounce handle_DATA()='250 SL E213 Unknown email ignored'

OBS_D2_DONE
```

### K.6 Script E — is_bounce truth table + transactional + iCloud

> **Labelling correction (read first):** In this script's source and print markers, **Part 3** is captioned "CANONICAL iCloud branch", but its two iCloud rows are produced by the module-level `handle()` called **directly** (`run_handle()`), which bypasses the `handle_DATA` -> `_handle` entry point. Those two rows are therefore **non-canonical (direct `handle()` call)**; the caption in the embedded source is inaccurate in that entry-point sense. The **canonical `handle_DATA`** confirmation of the identical iCloud outcomes (valid -> `250 SL E211 Bounce Forward phase handled`, invalid -> `550 SL E512 No such email log`) is in Phase **K.10** (`gb_run1.out`/`gb_run2.out`). Part 1 (`is_bounce` truth table) is already labelled non-canonical (direct calls); Part 2 (transactional) reports both `handle()` **and** `handle_DATA()` and its `handle_DATA()` column is canonical.

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_E_routing.py
```

**Observation-script source (`obs_E_routing.py`):**

```python
"""OBSERVATION SCRIPT E — Objective 6 (is_bounce spoofability) + routing completeness.
Part 1: is_bounce() truth table via DIRECT calls (NON-CANONICAL) over the two attacker-
controllable inputs (envelope.mail_from, message Content-Type).
Part 2: CANONICAL transactional branch: OOO->E206, bounce->E205, neither->VERPTransactional->E213.
Part 3: CANONICAL iCloud branch (mail_from=bounce+id+@domain, rcpt=alias; NO is_bounce gate)."""
import os, asyncio
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from aiosmtpd.smtp import Envelope
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email
from tests.utils import create_new_user

def msg_ct(ct, extra=""):
    return email.message_from_string("From: x@y.com\nTo: probe@sl.local\nSubject: s\n%sContent-Type: %s\n\nbody\n" % (extra, ct))

def run_handle(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    try: return "RETURNED %r" % (email_handler.handle(env, email.message_from_string(raw)),)
    except Exception as e: return "RAISED %s(%s)" % (type(e).__name__, e)

def run_handle_data(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]; env.original_content = raw.encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

transaction = connection.begin()
with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    try:
        print("===== PART 1: is_bounce() truth table [email_handler.py:L1813-L1818] (NON-CANONICAL direct calls) =====")
        for mf in ["<>", "attacker@evil.example"]:
            for ct in ["multipart/report", "text/plain"]:
                env = Envelope(); env.mail_from = mf; env.rcpt_tos = ["x@sl.local"]
                m = msg_ct(ct)
                print("is_bounce(mail_from=%-22r, Content-Type=%-18r) = %s"
                      % (mf, ct, email_handler.is_bounce(env, m)))

        print("\n===== PART 2: CANONICAL transactional branch (generate_verp_email(VerpType.transactional, id)) =====")
        taddr = generate_verp_email(VerpType.transactional, 1)
        OOO = "From: x@y.com\nTo: probe@sl.local\nSubject: ooo\nAuto-Submitted: auto-replied\nContent-Type: text/plain\n\nI am away\n"
        BOUNCE = ('From: MAILER-DAEMON@evil.example\nTo: probe@sl.local\nSubject: b\n'
                  'Content-Type: multipart/report; report-type=delivery-status; boundary="b"\n\n'
                  '--b\nContent-Type: text/plain\n\nfail\n--b\nContent-Type: message/delivery-status\n\nStatus: 5.1.1\n--b--\n')
        NONB = "From: x@y.com\nTo: probe@sl.local\nSubject: s\nContent-Type: text/plain\n\nhi\n"
        print("transactional addr = %r" % taddr)
        print("OOO (Auto-Submitted) -> handle()=%s | handle_DATA()=%r" % (run_handle("x@y.com", taddr, OOO), run_handle_data("x@y.com", taddr, OOO)))
        print("BOUNCE (<>,mp/report)-> handle()=%s | handle_DATA()=%r" % (run_handle("<>", taddr, BOUNCE), run_handle_data("<>", taddr, BOUNCE)))
        print("NEITHER (non-bounce) -> handle()=%s | handle_DATA()=%r" % (run_handle("x@y.com", taddr, NONB), run_handle_data("x@y.com", taddr, NONB)))

        print("\n===== PART 3: CANONICAL iCloud branch (mail_from=bounce+id+@sl.local, rcpt=alias; no is_bounce gate) =====")
        user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
        contact = Contact.create(user_id=user.id, alias_id=alias.id, website_email="c@example.com", reply_email="rep@sl.local", commit=True)
        el = EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=False, commit=True)
        VID = el.id; INV = 99999999999999
        for label, elid in [("VALID id", VID), ("INVALID id", INV)]:
            mf = "%s%s%s" % (config.BOUNCE_PREFIX, elid, config.BOUNCE_SUFFIX)
            print("iCloud %s: mail_from=%r rcpt=%r (non-bounce msg) -> handle()=%s"
                  % (label, mf, alias.email, run_handle(mf, alias.email, NONB)))
    finally:
        transaction.rollback(); Session.rollback(); Session.close()
print("\nOBS_E_DONE")
```

**Complete verbatim transcript (`obs_E.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/frysaxdvaqkipbhialfy
Upload files to local dir
>>> init logging <<<
2026-07-13 18:25:00,729 - SL - DEBUG - 3010 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-13 18:25:01,884 - SL - DEBUG - 3010 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 18:25:01,885 - SL - DEBUG - 3010 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 18:25:01,886 - SL - DEBUG - 3010 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:25:01,886 - SL - DEBUG - 3010 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
===== PART 1: is_bounce() truth table [email_handler.py:L1813-L1818] (NON-CANONICAL direct calls) =====
is_bounce(mail_from='<>'                  , Content-Type='multipart/report') = True
is_bounce(mail_from='<>'                  , Content-Type='text/plain'      ) = False
is_bounce(mail_from='attacker@evil.example', Content-Type='multipart/report') = False
is_bounce(mail_from='attacker@evil.example', Content-Type='text/plain'      ) = False

===== PART 2: CANONICAL transactional branch (generate_verp_email(VerpType.transactional, id)) =====
transactional addr = 'sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'
2026-07-13 18:25:01,908 - SL - INFO - 3010 - "/work/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:25:01,909 - SL - DEBUG - 3010 - "/work/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-13 18:25:01,910 - SL - DEBUG - 3010 - "/work/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:x@y.com, rcpt_tos:['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], header_from:x@y.com, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'x@y.com'), ('To', 'probe@sl.local'), ('Subject', 'ooo'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:25:01,914 - SL - DEBUG - 3010 - "/work/email_handler.py:1803" - is_automatic_out_of_office() -  - out-of-office email Auto-Submitted:auto-replied
2026-07-13 18:25:01,914 - SL - DEBUG - 3010 - "/work/email_handler.py:2049" - handle() -  - Ignore out-of-office for transactional emails. Headers: <bound method Message.items of <email.message.Message object at 0x78bc4bfd1f30>>
2026-07-13 18:25:01,916 - SL - DEBUG - 3010 - "/work/app/log.py:24" - set_message_id() -  - set message_id 707b1f9d-d9a8-433f-afa8-2e43cac1df7f
2026-07-13 18:25:01,916 - SL - DEBUG - 3010 - "/work/email_handler.py:2342" - _handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - ====>=====>====>====>====>====>====>====>
2026-07-13 18:25:01,916 - SL - INFO - 3010 - "/work/email_handler.py:2343" - _handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - New message, mail from x@y.com, rctp tos ['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local']
2026-07-13 18:25:01,916 - SL - INFO - 3010 - "/work/email_handler.py:1956" - handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:25:01,916 - SL - DEBUG - 3010 - "/work/email_handler.py:1963" - handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - Cannot parse Postfix queue ID from None None
2026-07-13 18:25:01,917 - SL - DEBUG - 3010 - "/work/email_handler.py:1980" - handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - ==>> Handle mail_from:x@y.com, rcpt_tos:['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], header_from:x@y.com, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'x@y.com'), ('To', 'probe@sl.local'), ('Subject', 'ooo'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:25:01,919 - SL - DEBUG - 3010 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - out-of-office email Auto-Submitted:auto-replied
2026-07-13 18:25:01,919 - SL - DEBUG - 3010 - "/work/email_handler.py:2049" - handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - Ignore out-of-office for transactional emails. Headers: <bound method Message.items of <email.message.Message object at 0x78bc4be04280>>
2026-07-13 18:25:01,919 - SL - INFO - 3010 - "/work/email_handler.py:2367" - _handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - Finish mail_from x@y.com, rcpt_tos ['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], takes 0.003751993179321289 seconds with return code '250 SL E206 Out of office'<<===
OOO (Auto-Submitted) -> handle()=RETURNED '250 SL E206 Out of office' | handle_DATA()='250 SL E206 Out of office'
2026-07-13 18:25:01,920 - SL - INFO - 3010 - "/work/email_handler.py:1956" - handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:25:01,920 - SL - DEBUG - 3010 - "/work/email_handler.py:1963" - handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - Cannot parse Postfix queue ID from None None
2026-07-13 18:25:01,921 - SL - DEBUG - 3010 - "/work/email_handler.py:1980" - handle() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'b'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:25:01,924 - SL - DEBUG - 3010 - "/work/email_handler.py:1824" - handle_transactional_bounce() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - handle transactional bounce sent to sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local
2026-07-13 18:25:01,925 - SL - INFO - 3010 - "/work/email_handler.py:1834" - handle_transactional_bounce() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - No transactional record for <> -> ['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local']
2026-07-13 18:25:01,926 - SL - DEBUG - 3010 - "/work/app/log.py:24" - set_message_id() - 707b1f9d-d9a8-433f-afa8-2e43cac1df7f - set message_id 3c770209-4c13-46b0-9abd-ac2d773e0353
2026-07-13 18:25:01,926 - SL - DEBUG - 3010 - "/work/email_handler.py:2342" - _handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:25:01,926 - SL - INFO - 3010 - "/work/email_handler.py:2343" - _handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - New message, mail from <>, rctp tos ['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local']
2026-07-13 18:25:01,927 - SL - INFO - 3010 - "/work/email_handler.py:1956" - handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:25:01,927 - SL - DEBUG - 3010 - "/work/email_handler.py:1963" - handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - Cannot parse Postfix queue ID from None None
2026-07-13 18:25:01,927 - SL - DEBUG - 3010 - "/work/email_handler.py:1980" - handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'b'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:25:01,929 - SL - DEBUG - 3010 - "/work/email_handler.py:1824" - handle_transactional_bounce() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - handle transactional bounce sent to sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local
2026-07-13 18:25:01,930 - SL - INFO - 3010 - "/work/email_handler.py:1834" - handle_transactional_bounce() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - No transactional record for <> -> ['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local']
2026-07-13 18:25:01,930 - SL - INFO - 3010 - "/work/email_handler.py:2367" - _handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - Finish mail_from <>, rcpt_tos ['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], takes 0.003950595855712891 seconds with return code '250 SL E205 bounce handled'<<===
BOUNCE (<>,mp/report)-> handle()=RETURNED '250 SL E205 bounce handled' | handle_DATA()='250 SL E205 bounce handled'
2026-07-13 18:25:01,930 - SL - INFO - 3010 - "/work/email_handler.py:1956" - handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:25:01,930 - SL - DEBUG - 3010 - "/work/email_handler.py:1963" - handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - Cannot parse Postfix queue ID from None None
2026-07-13 18:25:01,931 - SL - DEBUG - 3010 - "/work/email_handler.py:1980" - handle() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - ==>> Handle mail_from:x@y.com, rcpt_tos:['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], header_from:x@y.com, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'x@y.com'), ('To', 'probe@sl.local'), ('Subject', 's'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:25:01,933 - SL - DEBUG - 3010 - "/work/app/log.py:24" - set_message_id() - 3c770209-4c13-46b0-9abd-ac2d773e0353 - set message_id a850d46b-aa69-4479-ae30-c45ebf271c4d
2026-07-13 18:25:01,933 - SL - DEBUG - 3010 - "/work/email_handler.py:2342" - _handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - ====>=====>====>====>====>====>====>====>
2026-07-13 18:25:01,933 - SL - INFO - 3010 - "/work/email_handler.py:2343" - _handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - New message, mail from x@y.com, rctp tos ['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local']
2026-07-13 18:25:01,934 - SL - INFO - 3010 - "/work/email_handler.py:1956" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:25:01,934 - SL - DEBUG - 3010 - "/work/email_handler.py:1963" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Cannot parse Postfix queue ID from None None
2026-07-13 18:25:01,935 - SL - DEBUG - 3010 - "/work/email_handler.py:1980" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - ==>> Handle mail_from:x@y.com, rcpt_tos:['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], header_from:x@y.com, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'x@y.com'), ('To', 'probe@sl.local'), ('Subject', 's'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:25:01,937 - SL - WARNING - 3010 - "/work/email_handler.py:2309" - handle_DATA() - a850d46b-aa69-4479-ae30-c45ebf271c4d - email handling fail with error:VERPTransactional  mail_from:x@y.com, rcpt_tos:['sl.lmzcyibrfqqdemzygi4dmnk5.gc2uwfz7zbn2y@sl.local'], header_from:x@y.com, header_to:probe@sl.local
NEITHER (non-bounce) -> handle()=RAISED VERPTransactional(VERPTransactional ) | handle_DATA()='250 SL E213 Unknown email ignored'

===== PART 3: CANONICAL iCloud branch (mail_from=bounce+id+@sl.local, rcpt=alias; no is_bounce gate) =====
2026-07-13 18:25:02,216 - SL - INFO - 3010 - "/work/app/events/event_dispatcher.py:62" - send_event() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:02,230 - SL - DEBUG - 3010 - "/work/app/models.py:1459" - generate_random_alias_email() - a850d46b-aa69-4479-ae30-c45ebf271c4d - generate email weeper_became557@sl.local
2026-07-13 18:25:02,239 - SL - INFO - 3010 - "/work/app/events/event_dispatcher.py:62" - send_event() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:02,261 - SL - INFO - 3010 - "/work/email_handler.py:1956" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:25:02,261 - SL - DEBUG - 3010 - "/work/email_handler.py:1963" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Cannot parse Postfix queue ID from None None
2026-07-13 18:25:02,262 - SL - DEBUG - 3010 - "/work/email_handler.py:1980" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - ==>> Handle mail_from:bounce+397+@sl.local, rcpt_tos:['weeper_became557@sl.local'], header_from:x@y.com, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'x@y.com'), ('To', 'probe@sl.local'), ('Subject', 's'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:25:02,271 - SL - WARNING - 3010 - "/work/email_handler.py:2110" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - iCloud bounces <EmailLog 397> <Alias 842 weeper_became557@sl.local>, saved to
2026-07-13 18:25:02,272 - SL - DEBUG - 3010 - "/work/email_handler.py:1862" - handle_bounce() - a850d46b-aa69-4479-ae30-c45ebf271c4d - handle bounce for <EmailLog 397>, phase=forward, contact=<Contact 163 c@example.com 842>, alias=<Alias 842 weeper_became557@sl.local>
2026-07-13 18:25:02,273 - SL - ERROR - 3010 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Use <Alias 842 weeper_became557@sl.local> default mailbox <Mailbox 569 user_sc4x4df153@mailbox.test>
NoneType: None
2026-07-13 18:25:02,274 - SL - WARNING - 3010 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - a850d46b-aa69-4479-ae30-c45ebf271c4d - cannot get bounce info, debug at
2026-07-13 18:25:02,276 - SL - DEBUG - 3010 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Handle forward bounce <Contact 163 c@example.com 842> -> <Alias 842 weeper_became557@sl.local> -> <Mailbox 569 user_sc4x4df153@mailbox.test>. <EmailLog 397>
2026-07-13 18:25:02,281 - SL - WARNING - 3010 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Cannot parse original message from bounce message <Alias 842 weeper_became557@sl.local> <User 489 Test User user_sc4x4df153@mailbox.test> <Contact 163 c@example.com 842> refused-emails/full-81b6fb67-31c2-4dbe-a787-69ca9f2ffc91.eml
2026-07-13 18:25:02,284 - SL - DEBUG - 3010 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Create refused email <Refused Email 44 None 2026-07-20T18:25:02.283907+00:00>
2026-07-13 18:25:02,295 - SL - DEBUG - 3010 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Inform user <User 489 Test User user_sc4x4df153@mailbox.test> about a bounce from contact <Contact 163 c@example.com 842> to alias <Alias 842 weeper_became557@sl.local>
2026-07-13 18:25:02,326 - SL - DEBUG - 3010 - "/work/app/email_utils.py:303" - send_email() - a850d46b-aa69-4479-ae30-c45ebf271c4d - send email to user_sc4x4df153@mailbox.test, subject 'An email sent to weeper_became557@sl.local cannot be delivered to your mailbox'
2026-07-13 18:25:02,333 - SL - DEBUG - 3010 - "/work/app/mail_sender.py:131" - send() - a850d46b-aa69-4479-ae30-c45ebf271c4d - send email with subject 'An email sent to weeper_became557@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_sc4x4df153@mailbox.test'
iCloud VALID id: mail_from='bounce+397+@sl.local' rcpt='weeper_became557@sl.local' (non-bounce msg) -> handle()=RETURNED '250 SL E211 Bounce Forward phase handled'
2026-07-13 18:25:02,335 - SL - INFO - 3010 - "/work/email_handler.py:1956" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:25:02,336 - SL - DEBUG - 3010 - "/work/email_handler.py:1963" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - Cannot parse Postfix queue ID from None None
2026-07-13 18:25:02,337 - SL - DEBUG - 3010 - "/work/email_handler.py:1980" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - ==>> Handle mail_from:bounce+99999999999999+@sl.local, rcpt_tos:['weeper_became557@sl.local'], header_from:x@y.com, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'x@y.com'), ('To', 'probe@sl.local'), ('Subject', 's'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:25:02,349 - SL - WARNING - 3010 - "/work/email_handler.py:2110" - handle() - a850d46b-aa69-4479-ae30-c45ebf271c4d - iCloud bounces None <Alias 842 weeper_became557@sl.local>, saved to
2026-07-13 18:25:02,349 - SL - WARNING - 3010 - "/work/email_handler.py:1857" - handle_bounce() - a850d46b-aa69-4479-ae30-c45ebf271c4d - No such email log
iCloud INVALID id: mail_from='bounce+99999999999999+@sl.local' rcpt='weeper_became557@sl.local' (non-bounce msg) -> handle()=RETURNED '550 SL E512 No such email log'

OBS_E_DONE
```

### K.7 Script F — committed state transition on the bounce path

> **Labelling correction (read first):** This script's docstring calls the scenario "a mutating CANONICAL scenario", but the transition is driven by the module-level `handle()` called **directly** (`email_handler.handle(env, ...)`), which bypasses the `handle_DATA` -> `_handle` entry point. The `BEFORE`/`AFTER`/`DELTA` evidence below is therefore **non-canonical (direct `handle()` call)**; it is retained here verbatim for completeness. The **canonical `handle_DATA`** capture of the same `bounced`/`refused_email_id`/`bounced_mailbox_id` transition and the `+1`/`+1` `Bounce`/`RefusedEmail` growth (run twice) is in Phase **K.10** (`gb_run1.out`/`gb_run2.out`) and is quoted in Phase D.4.

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_F_state.py
```

**Observation-script source (`obs_F_state.py`):**

```python
"""OBSERVATION SCRIPT F — state transition for a mutating CANONICAL scenario:
a forward-phase bounce (valid id, active user, mail_from=<>, multipart/report) via handle().
Captures BEFORE (pre-handler) and AFTER (post-handler, re-read from the DB) for
EmailLog.bounced/.refused_email_id/.bounced_mailbox_id and the Bounce & RefusedEmail row counts.
NOTE: writes land in the EPHEMERAL in-container test DB (postgresql://test@localhost:15432/test);
the git repository is untouched. Verified empirically that Session.commit() persists (SQLAlchemy
1.3.24) and the flask_client fixture's rollback does NOT revert committed rows, so this reports the
real committed transition rather than claiming a reversal."""
import os
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from aiosmtpd.smtp import Envelope
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType, Bounce, RefusedEmail
from app.email_utils import generate_verp_email
from tests.utils import create_new_user

BOUNCE = ('From: MAILER-DAEMON@evil.example\nTo: probe@sl.local\nSubject: b\n'
          'Content-Type: multipart/report; report-type=delivery-status; boundary="b"\n\n'
          '--b\nContent-Type: text/plain\n\nfail\n--b\nContent-Type: message/delivery-status\n\nStatus: 5.1.1\n--b--\n')

def counts():
    return Session.query(Bounce).count(), Session.query(RefusedEmail).count()

with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
    contact = Contact.create(user_id=user.id, alias_id=alias.id, website_email="c@example.com", reply_email="rep@sl.local", commit=True)
    el = EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=False, commit=True)
    VID = el.id
    b0, r0 = counts()
    print("BEFORE : email_log(id=%s).bounced=%s refused_email_id=%s bounced_mailbox_id=%s | Bounce_rows=%s RefusedEmail_rows=%s"
          % (VID, el.bounced, el.refused_email_id, el.bounced_mailbox_id, b0, r0))

    env = Envelope(); env.mail_from = "<>"; env.rcpt_tos = [generate_verp_email(VerpType.bounce_forward, VID)]
    result = email_handler.handle(env, email.message_from_string(BOUNCE))
    print("HANDLER: handle() returned %r" % (result,))

    # AFTER: re-read from the DB via a fresh identity (expire then get)
    Session.expire_all()
    el_a = EmailLog.get(VID); b1, r1 = counts()
    print("AFTER  : email_log(id=%s).bounced=%s refused_email_id=%s bounced_mailbox_id=%s | Bounce_rows=%s RefusedEmail_rows=%s"
          % (VID, el_a.bounced, el_a.refused_email_id, el_a.bounced_mailbox_id, b1, r1))
    print("DELTA  : Bounce_rows +%d, RefusedEmail_rows +%d (committed to ephemeral test DB; repo untouched)"
          % (b1 - b0, r1 - r0))
print("\nOBS_F_DONE")
```

**Complete verbatim transcript (`obs_F.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/fpyrixphtfhkcuxxefpx
Upload files to local dir
>>> init logging <<<
2026-07-13 18:28:44,302 - SL - DEBUG - 3153 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-13 18:28:45,530 - SL - DEBUG - 3153 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 18:28:45,531 - SL - DEBUG - 3153 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 18:28:45,532 - SL - DEBUG - 3153 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:28:45,533 - SL - DEBUG - 3153 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:28:45,855 - SL - INFO - 3153 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:28:45,874 - SL - DEBUG - 3153 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email canoed_gypped736@sl.local
2026-07-13 18:28:45,885 - SL - INFO - 3153 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
BEFORE : email_log(id=400).bounced=False refused_email_id=None bounced_mailbox_id=None | Bounce_rows=29 RefusedEmail_rows=30
2026-07-13 18:28:45,913 - SL - INFO - 3153 - "/work/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:28:45,913 - SL - DEBUG - 3153 - "/work/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-13 18:28:45,915 - SL - DEBUG - 3153 - "/work/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibugaycyibsgm4deobwhboq.jsi3xh26psbhw@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'b'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:28:45,920 - SL - DEBUG - 3153 - "/work/email_handler.py:1862" - handle_bounce() -  - handle bounce for <EmailLog 400>, phase=forward, contact=<Contact 166 c@example.com 854>, alias=<Alias 854 canoed_gypped736@sl.local>
2026-07-13 18:28:45,924 - SL - ERROR - 3153 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() -  - Use <Alias 854 canoed_gypped736@sl.local> default mailbox <Mailbox 578 user_2ljmcw3jm5@mailbox.test>
NoneType: None
2026-07-13 18:28:45,925 - SL - WARNING - 3153 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() -  - cannot get bounce info, debug at
2026-07-13 18:28:45,927 - SL - DEBUG - 3153 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() -  - Handle forward bounce <Contact 166 c@example.com 854> -> <Alias 854 canoed_gypped736@sl.local> -> <Mailbox 578 user_2ljmcw3jm5@mailbox.test>. <EmailLog 400>
2026-07-13 18:28:45,932 - SL - WARNING - 3153 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() -  - Cannot parse original message from bounce message <Alias 854 canoed_gypped736@sl.local> <User 498 Test User user_2ljmcw3jm5@mailbox.test> <Contact 166 c@example.com 854> refused-emails/full-a5ef8435-a89b-4320-99e8-8e698004e876.eml
2026-07-13 18:28:45,934 - SL - DEBUG - 3153 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() -  - Create refused email <Refused Email 47 None 2026-07-20T18:28:45.934208+00:00>
2026-07-13 18:28:45,944 - SL - DEBUG - 3153 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() -  - Inform user <User 498 Test User user_2ljmcw3jm5@mailbox.test> about a bounce from contact <Contact 166 c@example.com 854> to alias <Alias 854 canoed_gypped736@sl.local>
2026-07-13 18:28:45,973 - SL - DEBUG - 3153 - "/work/app/email_utils.py:303" - send_email() -  - send email to user_2ljmcw3jm5@mailbox.test, subject 'An email sent to canoed_gypped736@sl.local cannot be delivered to your mailbox'
2026-07-13 18:28:45,978 - SL - DEBUG - 3153 - "/work/app/mail_sender.py:131" - send() -  - send email with subject 'An email sent to canoed_gypped736@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_2ljmcw3jm5@mailbox.test'
HANDLER: handle() returned '250 SL E211 Bounce Forward phase handled'
AFTER  : email_log(id=400).bounced=True refused_email_id=47 bounced_mailbox_id=578 | Bounce_rows=30 RefusedEmail_rows=31
DELTA  : Bounce_rows +1, RefusedEmail_rows +1 (committed to ephemeral test DB; repo untouched)

OBS_F_DONE
```

### K.8 Script H — iCloud signed mail_from fall-through (E200 + mail_from[0] bug)

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_H_icloud_signed.py
```

**Observation-script source (`obs_H_icloud_signed.py`):**

```python
"""OBSERVATION SCRIPT H — iCloud branch with a SIGNED mail_from: demonstrate the mail_from[0]
(first-character) handling at email_handler.py:L2101 so a signed address cannot satisfy the
verp_info branch, and capture the canonical wire response. Also confirms the E200 string.
Alias email captured to a plain str BEFORE driving the handler to avoid ORM detachment on repeat."""
import os, asyncio
os.environ["CONFIG"] = "/work/tests/test.env"
from aiosmtpd.smtp import Envelope
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.email import status
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email
from tests.utils import create_new_user

def handle_data(mf, rcpt):
    env = Envelope(); env.mail_from = mf; env.rcpt_tos = [rcpt]
    env.original_content = ("From: attacker@evil.example\nTo: %s\nSubject: hi\nContent-Type: text/plain\n\nhello\n" % rcpt).encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
    contact = Contact.create(user_id=user.id, alias_id=alias.id, website_email="c@example.com", reply_email="rep@sl.local", commit=True)
    el = EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=False, commit=True)
    VID = el.id
    alias_email = str(alias.email)   # capture to plain str BEFORE handler expires the Session
    signed_mf = generate_verp_email(VerpType.bounce_forward, VID)
    print("status.E200 literal string = %r" % status.E200)
    print("FIXTURE valid_id=%s alias=%s" % (VID, alias_email))
    print("signed mail_from = %r  (mail_from[0]=%r, the FIRST CHARACTER, per email_handler.py:L2101)" % (signed_mf, signed_mf[0]))
    r1 = handle_data(signed_mf, alias_email)
    print("iCloud SIGNED mail_from -> handle_DATA() (run1) = %r" % r1)
    r2 = handle_data(signed_mf, alias_email)
    print("iCloud SIGNED mail_from -> handle_DATA() (run2) = %r" % r2)
    print("STABLE=%s" % (r1 == r2))
print("OBS_H_DONE")
```

**Complete verbatim transcript (`obs_H.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ttipwdgcljaadvpcjpaz
Upload files to local dir
>>> init logging <<<
2026-07-13 18:41:25,688 - SL - DEBUG - 3848 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-13 18:41:26,746 - SL - DEBUG - 3848 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 18:41:26,747 - SL - DEBUG - 3848 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 18:41:26,748 - SL - DEBUG - 3848 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:41:26,748 - SL - DEBUG - 3848 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:41:27,047 - SL - INFO - 3848 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:41:27,060 - SL - DEBUG - 3848 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email matrix_hailed351@sl.local
2026-07-13 18:41:27,067 - SL - INFO - 3848 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
status.E200 literal string = '250 Message accepted for delivery'
FIXTURE valid_id=416 alias=matrix_hailed351@sl.local
signed mail_from = 'sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local'  (mail_from[0]='s', the FIRST CHARACTER, per email_handler.py:L2101)
2026-07-13 18:41:27,086 - SL - DEBUG - 3848 - "/work/app/log.py:24" - set_message_id() -  - set message_id f8db22bb-9694-443d-8f91-913c78bc55fa
2026-07-13 18:41:27,086 - SL - DEBUG - 3848 - "/work/email_handler.py:2342" - _handle() - f8db22bb-9694-443d-8f91-913c78bc55fa - ====>=====>====>====>====>====>====>====>
2026-07-13 18:41:27,086 - SL - INFO - 3848 - "/work/email_handler.py:2343" - _handle() - f8db22bb-9694-443d-8f91-913c78bc55fa - New message, mail from sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local, rctp tos ['matrix_hailed351@sl.local']
2026-07-13 18:41:27,087 - SL - INFO - 3848 - "/work/email_handler.py:1956" - handle() - f8db22bb-9694-443d-8f91-913c78bc55fa - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:41:27,087 - SL - DEBUG - 3848 - "/work/email_handler.py:1963" - handle() - f8db22bb-9694-443d-8f91-913c78bc55fa - Cannot parse Postfix queue ID from None None
2026-07-13 18:41:27,088 - SL - DEBUG - 3848 - "/work/email_handler.py:1980" - handle() - f8db22bb-9694-443d-8f91-913c78bc55fa - ==>> Handle mail_from:sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local, rcpt_tos:['matrix_hailed351@sl.local'], header_from:attacker@evil.example, header_to:matrix_hailed351@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'matrix_hailed351@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:41:27,092 - SL - DEBUG - 3848 - "/work/email_handler.py:2202" - handle() - f8db22bb-9694-443d-8f91-913c78bc55fa - Forward phase sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local(attacker@evil.example) -> matrix_hailed351@sl.local
2026-07-13 18:41:27,101 - SL - DEBUG - 3848 - "/work/email_handler.py:580" - handle_forward() - f8db22bb-9694-443d-8f91-913c78bc55fa - Create or get contact for from_header:attacker@evil.example
2026-07-13 18:41:27,120 - SL - DEBUG - 3848 - "/work/app/contact_utils.py:110" - create_contact() - f8db22bb-9694-443d-8f91-913c78bc55fa - Created contact <Contact 179 attacker@evil.example 884> for alias <Alias 884 matrix_hailed351@sl.local> with email attacker@evil.example invalid_email=False
2026-07-13 18:41:27,120 - SL - INFO - 3848 - "/work/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - f8db22bb-9694-443d-8f91-913c78bc55fa - DMARC check disabled
2026-07-13 18:41:27,128 - SL - DEBUG - 3848 - "/work/email_handler.py:688" - forward_email_to_mailbox() - f8db22bb-9694-443d-8f91-913c78bc55fa - Forward <Contact 179 attacker@evil.example 884> -> <Alias 884 matrix_hailed351@sl.local> -> <Mailbox 593 user_3qb0wqd133@mailbox.test>
2026-07-13 18:41:27,130 - SL - DEBUG - 3848 - "/work/email_handler.py:740" - forward_email_to_mailbox() - f8db22bb-9694-443d-8f91-913c78bc55fa - Create <EmailLog 417> for <Contact 179 attacker@evil.example 884>, <User 513 Test User user_3qb0wqd133@mailbox.test>, <Mailbox 593 user_3qb0wqd133@mailbox.test>
2026-07-13 18:41:27,135 - SL - WARNING - 3848 - "/work/email_handler.py:857" - forward_email_to_mailbox() - f8db22bb-9694-443d-8f91-913c78bc55fa - missing date header, create one
2026-07-13 18:41:27,135 - SL - DEBUG - 3848 - "/work/email_handler.py:867" - forward_email_to_mailbox() - f8db22bb-9694-443d-8f91-913c78bc55fa - From header, new:"attacker at evil.example" <attacker_at_evil_example_zyeshrye@sl.local>, old:attacker@evil.example
2026-07-13 18:41:27,135 - SL - DEBUG - 3848 - "/work/email_handler.py:316" - replace_header_when_forward() - f8db22bb-9694-443d-8f91-913c78bc55fa - Delete Cc header, old value None
2026-07-13 18:41:27,135 - SL - DEBUG - 3848 - "/work/email_handler.py:313" - replace_header_when_forward() - f8db22bb-9694-443d-8f91-913c78bc55fa - Replace To header, old: matrix_hailed351@sl.local, new: matrix_hailed351@sl.local
2026-07-13 18:41:27,135 - SL - INFO - 3848 - "/work/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - f8db22bb-9694-443d-8f91-913c78bc55fa - Email has no unsubscribe header
2026-07-13 18:41:27,138 - SL - DEBUG - 3848 - "/work/email_handler.py:893" - forward_email_to_mailbox() - f8db22bb-9694-443d-8f91-913c78bc55fa - Forward mail from attacker@evil.example to user_3qb0wqd133@mailbox.test, mail_options:[], rcpt_options:[]
2026-07-13 18:41:27,138 - SL - DEBUG - 3848 - "/work/app/mail_sender.py:131" - send() - f8db22bb-9694-443d-8f91-913c78bc55fa - send email with subject 'hi', from '"attacker at evil.example" <attacker_at_evil_example_zyeshrye@sl.local>' to 'matrix_hailed351@sl.local'
2026-07-13 18:41:27,138 - SL - INFO - 3848 - "/work/email_handler.py:2367" - _handle() - f8db22bb-9694-443d-8f91-913c78bc55fa - Finish mail_from sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local, rcpt_tos ['matrix_hailed351@sl.local'], takes 0.051981210708618164 seconds with return code '250 Message accepted for delivery'<<===
iCloud SIGNED mail_from -> handle_DATA() (run1) = '250 Message accepted for delivery'
2026-07-13 18:41:27,139 - SL - DEBUG - 3848 - "/work/app/log.py:24" - set_message_id() - f8db22bb-9694-443d-8f91-913c78bc55fa - set message_id 9b58d7a3-d834-45e1-b904-e95a6744627d
2026-07-13 18:41:27,139 - SL - DEBUG - 3848 - "/work/email_handler.py:2342" - _handle() - 9b58d7a3-d834-45e1-b904-e95a6744627d - ====>=====>====>====>====>====>====>====>
2026-07-13 18:41:27,139 - SL - INFO - 3848 - "/work/email_handler.py:2343" - _handle() - 9b58d7a3-d834-45e1-b904-e95a6744627d - New message, mail from sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local, rctp tos ['matrix_hailed351@sl.local']
2026-07-13 18:41:27,140 - SL - INFO - 3848 - "/work/email_handler.py:1956" - handle() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:41:27,140 - SL - DEBUG - 3848 - "/work/email_handler.py:1963" - handle() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Cannot parse Postfix queue ID from None None
2026-07-13 18:41:27,140 - SL - DEBUG - 3848 - "/work/email_handler.py:1980" - handle() - 9b58d7a3-d834-45e1-b904-e95a6744627d - ==>> Handle mail_from:sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local, rcpt_tos:['matrix_hailed351@sl.local'], header_from:attacker@evil.example, header_to:matrix_hailed351@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'matrix_hailed351@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:41:27,145 - SL - DEBUG - 3848 - "/work/email_handler.py:2202" - handle() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Forward phase sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local(attacker@evil.example) -> matrix_hailed351@sl.local
2026-07-13 18:41:27,155 - SL - DEBUG - 3848 - "/work/email_handler.py:580" - handle_forward() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Create or get contact for from_header:attacker@evil.example
2026-07-13 18:41:27,156 - SL - INFO - 3848 - "/work/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 9b58d7a3-d834-45e1-b904-e95a6744627d - DMARC check disabled
2026-07-13 18:41:27,163 - SL - DEBUG - 3848 - "/work/email_handler.py:688" - forward_email_to_mailbox() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Forward <Contact 179 attacker@evil.example 884> -> <Alias 884 matrix_hailed351@sl.local> -> <Mailbox 593 user_3qb0wqd133@mailbox.test>
2026-07-13 18:41:27,165 - SL - DEBUG - 3848 - "/work/email_handler.py:740" - forward_email_to_mailbox() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Create <EmailLog 418> for <Contact 179 attacker@evil.example 884>, <User 513 Test User user_3qb0wqd133@mailbox.test>, <Mailbox 593 user_3qb0wqd133@mailbox.test>
2026-07-13 18:41:27,170 - SL - WARNING - 3848 - "/work/email_handler.py:857" - forward_email_to_mailbox() - 9b58d7a3-d834-45e1-b904-e95a6744627d - missing date header, create one
2026-07-13 18:41:27,170 - SL - DEBUG - 3848 - "/work/email_handler.py:867" - forward_email_to_mailbox() - 9b58d7a3-d834-45e1-b904-e95a6744627d - From header, new:"attacker at evil.example" <attacker_at_evil_example_zyeshrye@sl.local>, old:attacker@evil.example
2026-07-13 18:41:27,171 - SL - DEBUG - 3848 - "/work/email_handler.py:316" - replace_header_when_forward() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Delete Cc header, old value None
2026-07-13 18:41:27,171 - SL - DEBUG - 3848 - "/work/email_handler.py:313" - replace_header_when_forward() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Replace To header, old: matrix_hailed351@sl.local, new: matrix_hailed351@sl.local
2026-07-13 18:41:27,171 - SL - INFO - 3848 - "/work/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Email has no unsubscribe header
2026-07-13 18:41:27,173 - SL - DEBUG - 3848 - "/work/email_handler.py:893" - forward_email_to_mailbox() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Forward mail from attacker@evil.example to user_3qb0wqd133@mailbox.test, mail_options:[], rcpt_options:[]
2026-07-13 18:41:27,173 - SL - DEBUG - 3848 - "/work/app/mail_sender.py:131" - send() - 9b58d7a3-d834-45e1-b904-e95a6744627d - send email with subject 'hi', from '"attacker at evil.example" <attacker_at_evil_example_zyeshrye@sl.local>' to 'matrix_hailed351@sl.local'
2026-07-13 18:41:27,173 - SL - INFO - 3848 - "/work/email_handler.py:2367" - _handle() - 9b58d7a3-d834-45e1-b904-e95a6744627d - Finish mail_from sl.lmycyibuge3cyibsgm4deobygfoq.3kdbyr2fsmdky@sl.local, rcpt_tos ['matrix_hailed351@sl.local'], takes 0.034604549407958984 seconds with return code '250 Message accepted for delivery'<<===
iCloud SIGNED mail_from -> handle_DATA() (run2) = '250 Message accepted for delivery'
STABLE=True
OBS_H_DONE
```

### K.9 Script G — determinism (5x repetition of exact inputs)

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_G_determinism.py
```

**Observation-script source (`obs_G_determinism.py`):**

```python
"""OBSERVATION SCRIPT G — determinism: drive the SAME unchanged probe repeatedly through the
canonical path and report the observed distribution (addresses the 'run-to-run inconsistency'
concern). 5 iterations each for: invalid-id unsigned via _handle(); invalid-id signed via
_handle(); valid-id non-bounce via handle_DATA()."""
import os, asyncio, collections
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from aiosmtpd.smtp import Envelope
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email
from tests.utils import create_new_user

NONB = "From: attacker@evil.example\nTo: probe@sl.local\nSubject: hi\nContent-Type: text/plain\n\nhello\n"
def _handle(mf, rcpt):
    env = Envelope(); env.mail_from = mf; env.rcpt_tos = [rcpt]
    return email_handler.MailHandler()._handle(env, email.message_from_string(NONB))
def _handle_data(mf, rcpt):
    env = Envelope(); env.mail_from = mf; env.rcpt_tos = [rcpt]; env.original_content = NONB.encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    user = create_new_user(); alias = Alias.create_new_random(user); Session.commit()
    contact = Contact.create(user_id=user.id, alias_id=alias.id, website_email="c@example.com", reply_email="rep@sl.local", commit=True)
    el = EmailLog.create(user_id=user.id, contact_id=contact.id, alias_id=alias.id, is_reply=False, commit=True)
    VID = el.id; INV = 99999999999999
    old_inv = "%s%s%s" % (config.BOUNCE_PREFIX, INV, config.BOUNCE_SUFFIX)
    signed_inv = generate_verp_email(VerpType.bounce_forward, INV)
    old_val = "%s%s%s" % (config.BOUNCE_PREFIX, VID, config.BOUNCE_SUFFIX)
    print("FIXTURE valid_id=%s invalid_id=%s" % (VID, INV))
    for label, fn, mf, rcpt in [
        ("invalid unsigned _handle()", _handle, "attacker@evil.example", old_inv),
        ("invalid signed   _handle()", _handle, "attacker@evil.example", signed_inv),
        ("valid  unsigned  handle_DATA()", _handle_data, "attacker@evil.example", old_val),
    ]:
        results = [fn(mf, rcpt) for _ in range(5)]
        dist = collections.Counter(results)
        print("\n%s  x5:" % label)
        for i, r in enumerate(results, 1):
            print("   run %d: %r" % (i, r))
        print("   distribution: %s  -> %s" % (dict(dist), "DETERMINISTIC (5/5 identical)" if len(dist)==1 else "NON-DETERMINISTIC"))
print("\nOBS_G_DONE")
```

**Complete verbatim transcript (`obs_G.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/kvlzmhuvnghbyfdjqtgg
Upload files to local dir
>>> init logging <<<
2026-07-13 18:29:14,248 - SL - DEBUG - 3213 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-13 18:29:15,366 - SL - DEBUG - 3213 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 18:29:15,367 - SL - DEBUG - 3213 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 18:29:15,368 - SL - DEBUG - 3213 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:29:15,368 - SL - DEBUG - 3213 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:29:15,672 - SL - INFO - 3213 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:29:15,687 - SL - DEBUG - 3213 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email caulks_whinny943@sl.local
2026-07-13 18:29:15,695 - SL - INFO - 3213 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURE valid_id=402 invalid_id=99999999999999
2026-07-13 18:29:15,715 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() -  - set message_id 14a43870-31ff-4804-8499-0a4960a77e3a
2026-07-13 18:29:15,715 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 14a43870-31ff-4804-8499-0a4960a77e3a - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,715 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 14a43870-31ff-4804-8499-0a4960a77e3a - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:29:15,716 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 14a43870-31ff-4804-8499-0a4960a77e3a - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,716 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 14a43870-31ff-4804-8499-0a4960a77e3a - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,717 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 14a43870-31ff-4804-8499-0a4960a77e3a - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,723 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - 14a43870-31ff-4804-8499-0a4960a77e3a - No such email log
2026-07-13 18:29:15,723 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - 14a43870-31ff-4804-8499-0a4960a77e3a - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.0075724124908447266 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:29:15,723 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 14a43870-31ff-4804-8499-0a4960a77e3a - set message_id ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9
2026-07-13 18:29:15,723 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,723 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9 - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:29:15,724 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,724 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,725 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,740 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9 - No such email log
2026-07-13 18:29:15,740 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9 - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.016519546508789062 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:29:15,740 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - ea6e7c67-ca9f-4f73-b85e-7b943e0dbcd9 - set message_id 1188d3cd-53d3-4d75-9a41-d0b8654cc7da
2026-07-13 18:29:15,740 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 1188d3cd-53d3-4d75-9a41-d0b8654cc7da - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,740 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 1188d3cd-53d3-4d75-9a41-d0b8654cc7da - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:29:15,741 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 1188d3cd-53d3-4d75-9a41-d0b8654cc7da - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,741 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 1188d3cd-53d3-4d75-9a41-d0b8654cc7da - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,742 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 1188d3cd-53d3-4d75-9a41-d0b8654cc7da - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,746 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - 1188d3cd-53d3-4d75-9a41-d0b8654cc7da - No such email log
2026-07-13 18:29:15,746 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - 1188d3cd-53d3-4d75-9a41-d0b8654cc7da - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.005469083786010742 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:29:15,746 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 1188d3cd-53d3-4d75-9a41-d0b8654cc7da - set message_id 7d25b90a-95fa-40f4-8cd7-53dd77e05d26
2026-07-13 18:29:15,746 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 7d25b90a-95fa-40f4-8cd7-53dd77e05d26 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,746 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 7d25b90a-95fa-40f4-8cd7-53dd77e05d26 - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:29:15,747 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 7d25b90a-95fa-40f4-8cd7-53dd77e05d26 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,747 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 7d25b90a-95fa-40f4-8cd7-53dd77e05d26 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,748 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 7d25b90a-95fa-40f4-8cd7-53dd77e05d26 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,752 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - 7d25b90a-95fa-40f4-8cd7-53dd77e05d26 - No such email log
2026-07-13 18:29:15,752 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - 7d25b90a-95fa-40f4-8cd7-53dd77e05d26 - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.005541324615478516 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:29:15,752 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 7d25b90a-95fa-40f4-8cd7-53dd77e05d26 - set message_id 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074
2026-07-13 18:29:15,752 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,752 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074 - New message, mail from attacker@evil.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-13 18:29:15,753 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,753 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,754 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,758 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074 - No such email log
2026-07-13 18:29:15,758 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074 - Finish mail_from attacker@evil.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.005830526351928711 seconds with return code '550 SL E512 No such email log'<<===

invalid unsigned _handle()  x5:
   run 1: '550 SL E512 No such email log'
   run 2: '550 SL E512 No such email log'
   run 3: '550 SL E512 No such email log'
   run 4: '550 SL E512 No such email log'
   run 5: '550 SL E512 No such email log'
   distribution: {'550 SL E512 No such email log': 5}  -> DETERMINISTIC (5/5 identical)
2026-07-13 18:29:15,758 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 18ef81a5-3ee3-4eb3-a9f1-d881b07f6074 - set message_id f2ac4e4b-cc96-4104-8628-09ace712ada8
2026-07-13 18:29:15,759 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - f2ac4e4b-cc96-4104-8628-09ace712ada8 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,759 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - f2ac4e4b-cc96-4104-8628-09ace712ada8 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local']
2026-07-13 18:29:15,759 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - f2ac4e4b-cc96-4104-8628-09ace712ada8 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,759 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - f2ac4e4b-cc96-4104-8628-09ace712ada8 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,760 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - f2ac4e4b-cc96-4104-8628-09ace712ada8 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,764 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - f2ac4e4b-cc96-4104-8628-09ace712ada8 - No such email log
2026-07-13 18:29:15,764 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - f2ac4e4b-cc96-4104-8628-09ace712ada8 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], takes 0.0051767826080322266 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:29:15,764 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - f2ac4e4b-cc96-4104-8628-09ace712ada8 - set message_id 1253aee1-6aa9-47b6-b9ff-9103a6beedfa
2026-07-13 18:29:15,764 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 1253aee1-6aa9-47b6-b9ff-9103a6beedfa - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,764 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 1253aee1-6aa9-47b6-b9ff-9103a6beedfa - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local']
2026-07-13 18:29:15,765 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 1253aee1-6aa9-47b6-b9ff-9103a6beedfa - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,765 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 1253aee1-6aa9-47b6-b9ff-9103a6beedfa - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,766 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 1253aee1-6aa9-47b6-b9ff-9103a6beedfa - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,769 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - 1253aee1-6aa9-47b6-b9ff-9103a6beedfa - No such email log
2026-07-13 18:29:15,769 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - 1253aee1-6aa9-47b6-b9ff-9103a6beedfa - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], takes 0.005060434341430664 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:29:15,769 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 1253aee1-6aa9-47b6-b9ff-9103a6beedfa - set message_id fd6c810a-867b-4d58-8ba1-c116bfc2e563
2026-07-13 18:29:15,770 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - fd6c810a-867b-4d58-8ba1-c116bfc2e563 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,770 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - fd6c810a-867b-4d58-8ba1-c116bfc2e563 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local']
2026-07-13 18:29:15,770 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - fd6c810a-867b-4d58-8ba1-c116bfc2e563 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,770 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - fd6c810a-867b-4d58-8ba1-c116bfc2e563 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,771 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - fd6c810a-867b-4d58-8ba1-c116bfc2e563 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,775 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - fd6c810a-867b-4d58-8ba1-c116bfc2e563 - No such email log
2026-07-13 18:29:15,793 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - fd6c810a-867b-4d58-8ba1-c116bfc2e563 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], takes 0.02383255958557129 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:29:15,794 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - fd6c810a-867b-4d58-8ba1-c116bfc2e563 - set message_id 8cc89701-91bb-421c-b7c0-6bbf8e649959
2026-07-13 18:29:15,794 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 8cc89701-91bb-421c-b7c0-6bbf8e649959 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,794 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 8cc89701-91bb-421c-b7c0-6bbf8e649959 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local']
2026-07-13 18:29:15,795 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 8cc89701-91bb-421c-b7c0-6bbf8e649959 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,795 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 8cc89701-91bb-421c-b7c0-6bbf8e649959 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,796 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 8cc89701-91bb-421c-b7c0-6bbf8e649959 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,799 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - 8cc89701-91bb-421c-b7c0-6bbf8e649959 - No such email log
2026-07-13 18:29:15,799 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - 8cc89701-91bb-421c-b7c0-6bbf8e649959 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], takes 0.005406856536865234 seconds with return code '550 SL E512 No such email log'<<===
2026-07-13 18:29:15,800 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 8cc89701-91bb-421c-b7c0-6bbf8e649959 - set message_id fe2c850a-b25b-4864-b605-17b3fd32b7b1
2026-07-13 18:29:15,800 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - fe2c850a-b25b-4864-b605-17b3fd32b7b1 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,800 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - fe2c850a-b25b-4864-b605-17b3fd32b7b1 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local']
2026-07-13 18:29:15,800 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - fe2c850a-b25b-4864-b605-17b3fd32b7b1 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,800 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - fe2c850a-b25b-4864-b605-17b3fd32b7b1 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,801 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - fe2c850a-b25b-4864-b605-17b3fd32b7b1 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,805 - SL - WARNING - 3213 - "/work/email_handler.py:2066" - handle() - fe2c850a-b25b-4864-b605-17b3fd32b7b1 - No such email log
2026-07-13 18:29:15,805 - SL - INFO - 3213 - "/work/email_handler.py:2367" - _handle() - fe2c850a-b25b-4864-b605-17b3fd32b7b1 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmrygy4v2.iuqljl5chr3r4@sl.local'], takes 0.005127429962158203 seconds with return code '550 SL E512 No such email log'<<===

invalid signed   _handle()  x5:
   run 1: '550 SL E512 No such email log'
   run 2: '550 SL E512 No such email log'
   run 3: '550 SL E512 No such email log'
   run 4: '550 SL E512 No such email log'
   run 5: '550 SL E512 No such email log'
   distribution: {'550 SL E512 No such email log': 5}  -> DETERMINISTIC (5/5 identical)
2026-07-13 18:29:15,805 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - fe2c850a-b25b-4864-b605-17b3fd32b7b1 - set message_id 9e24258f-d09d-4114-bf15-cf2c7c4b2e3b
2026-07-13 18:29:15,806 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 9e24258f-d09d-4114-bf15-cf2c7c4b2e3b - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,806 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 9e24258f-d09d-4114-bf15-cf2c7c4b2e3b - New message, mail from attacker@evil.example, rctp tos ['bounce+402+@sl.local']
2026-07-13 18:29:15,806 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 9e24258f-d09d-4114-bf15-cf2c7c4b2e3b - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,806 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 9e24258f-d09d-4114-bf15-cf2c7c4b2e3b - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,807 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 9e24258f-d09d-4114-bf15-cf2c7c4b2e3b - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,811 - SL - WARNING - 3213 - "/work/email_handler.py:2309" - handle_DATA() - 9e24258f-d09d-4114-bf15-cf2c7c4b2e3b - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
2026-07-13 18:29:15,811 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 9e24258f-d09d-4114-bf15-cf2c7c4b2e3b - set message_id 14d2c867-1a93-458b-bb1f-9ee9825fa678
2026-07-13 18:29:15,811 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 14d2c867-1a93-458b-bb1f-9ee9825fa678 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,811 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 14d2c867-1a93-458b-bb1f-9ee9825fa678 - New message, mail from attacker@evil.example, rctp tos ['bounce+402+@sl.local']
2026-07-13 18:29:15,812 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 14d2c867-1a93-458b-bb1f-9ee9825fa678 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,812 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 14d2c867-1a93-458b-bb1f-9ee9825fa678 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,813 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 14d2c867-1a93-458b-bb1f-9ee9825fa678 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,816 - SL - WARNING - 3213 - "/work/email_handler.py:2309" - handle_DATA() - 14d2c867-1a93-458b-bb1f-9ee9825fa678 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
2026-07-13 18:29:15,817 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 14d2c867-1a93-458b-bb1f-9ee9825fa678 - set message_id 87c2dc43-1c48-4259-896c-1a92e14ec0d5
2026-07-13 18:29:15,817 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 87c2dc43-1c48-4259-896c-1a92e14ec0d5 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,817 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 87c2dc43-1c48-4259-896c-1a92e14ec0d5 - New message, mail from attacker@evil.example, rctp tos ['bounce+402+@sl.local']
2026-07-13 18:29:15,818 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 87c2dc43-1c48-4259-896c-1a92e14ec0d5 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,818 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 87c2dc43-1c48-4259-896c-1a92e14ec0d5 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,818 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 87c2dc43-1c48-4259-896c-1a92e14ec0d5 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,822 - SL - WARNING - 3213 - "/work/email_handler.py:2309" - handle_DATA() - 87c2dc43-1c48-4259-896c-1a92e14ec0d5 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
2026-07-13 18:29:15,822 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 87c2dc43-1c48-4259-896c-1a92e14ec0d5 - set message_id 51e5979f-8aff-434e-b133-6a9bc979f48d
2026-07-13 18:29:15,822 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 51e5979f-8aff-434e-b133-6a9bc979f48d - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,822 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 51e5979f-8aff-434e-b133-6a9bc979f48d - New message, mail from attacker@evil.example, rctp tos ['bounce+402+@sl.local']
2026-07-13 18:29:15,823 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 51e5979f-8aff-434e-b133-6a9bc979f48d - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,823 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 51e5979f-8aff-434e-b133-6a9bc979f48d - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,824 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 51e5979f-8aff-434e-b133-6a9bc979f48d - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,827 - SL - WARNING - 3213 - "/work/email_handler.py:2309" - handle_DATA() - 51e5979f-8aff-434e-b133-6a9bc979f48d - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local
2026-07-13 18:29:15,827 - SL - DEBUG - 3213 - "/work/app/log.py:24" - set_message_id() - 51e5979f-8aff-434e-b133-6a9bc979f48d - set message_id 5b7789c5-6823-4a0a-97c1-8a0368a3aba4
2026-07-13 18:29:15,827 - SL - DEBUG - 3213 - "/work/email_handler.py:2342" - _handle() - 5b7789c5-6823-4a0a-97c1-8a0368a3aba4 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:29:15,828 - SL - INFO - 3213 - "/work/email_handler.py:2343" - _handle() - 5b7789c5-6823-4a0a-97c1-8a0368a3aba4 - New message, mail from attacker@evil.example, rctp tos ['bounce+402+@sl.local']
2026-07-13 18:29:15,828 - SL - INFO - 3213 - "/work/email_handler.py:1956" - handle() - 5b7789c5-6823-4a0a-97c1-8a0368a3aba4 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 18:29:15,828 - SL - DEBUG - 3213 - "/work/email_handler.py:1963" - handle() - 5b7789c5-6823-4a0a-97c1-8a0368a3aba4 - Cannot parse Postfix queue ID from None None
2026-07-13 18:29:15,829 - SL - DEBUG - 3213 - "/work/email_handler.py:1980" - handle() - 5b7789c5-6823-4a0a-97c1-8a0368a3aba4 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 18:29:15,832 - SL - WARNING - 3213 - "/work/email_handler.py:2309" - handle_DATA() - 5b7789c5-6823-4a0a-97c1-8a0368a3aba4 - email handling fail with error:VERPForward  mail_from:attacker@evil.example, rcpt_tos:['bounce+402+@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local

valid  unsigned  handle_DATA()  x5:
   run 1: '250 SL E213 Unknown email ignored'
   run 2: '250 SL E213 Unknown email ignored'
   run 3: '250 SL E213 Unknown email ignored'
   run 4: '250 SL E213 Unknown email ignored'
   run 5: '250 SL E213 Unknown email ignored'
   distribution: {'250 SL E213 Unknown email ignored': 5}  -> DETERMINISTIC (5/5 identical)

OBS_G_DONE
```


### K.10 Script GB — canonical Group-B addendum (OUT-OF-OFFICE matrix, iCloud & state via handle_DATA)

This appendix supplies the **canonical `handle_DATA()` evidence** for the out-of-office matrix (Phase C.3/C.4, Objective 2), the iCloud branch (Phase C.5), and the committed state transition (Phase D.4). Every value below is produced by the real inbound entry point `MailHandler.handle_DATA()` -> `_handle()` -> `handle()` — **not** a direct `handle()` call. It supersedes, as the canonical source, the direct-`handle()` iCloud rows in K.6 Part 3 and the direct-`handle()` state rows in K.7 (both of which are retained verbatim but relabelled *(non-canonical, direct call)* in their sections). The script was run **twice**; the OUTCOME SUMMARY blocks are byte-identical across the two runs (SHA-256 shown below).

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_group_b.py
```

**Observation-script source (`obs_group_b.py`):**

```python
"""OBSERVATION SCRIPT (temporary, non-repository) — Group B CANONICAL addendum.
Drives the REAL inbound path MailHandler.handle_DATA() -> _handle() -> handle() for:
  PART A: forward & reply OUT-OF-OFFICE matrix (OLD unsigned + signed x valid + invalid id),
          OOO = 'Auto-Submitted: auto-replied' header, text/plain, non-<> sender (is_bounce False,
          is_automatic_out_of_office True) -> observe the fall-through outcome [L2071-2072 / L2092-2093].
  PART B: iCloud branch via handle_DATA (bounce addr in envelope mail_from, alias in rcpt, non-bounce
          body; no is_bounce gate) -> observe valid (E211) vs invalid (E512).
  PART C: committed STATE TRANSITION via handle_DATA: capture EmailLog.bounced / refused_email_id /
          bounced_mailbox_id and Bounce / RefusedEmail row counts BEFORE and AFTER a genuine
          forward-bounce handle_DATA call (mail_from='<>', signed bounce_forward, multipart/report).
Every outcome is produced by handle_DATA (canonical). Prints a compact OUTCOME SUMMARY for run
comparison. Only handle_DATA is invoked (no direct helper calls for observed values)."""
import os, asyncio, arrow, sys
os.environ["CONFIG"] = "/work/tests/test.env"
import email
from aiosmtpd.smtp import Envelope
import aiosmtpd
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType, Bounce, RefusedEmail
from app.email_utils import generate_verp_email
from tests.utils import create_new_user

OOO = ("From: postmaster@corp.example\r\nTo: probe@sl.local\r\nSubject: Away\r\n"
       "Auto-Submitted: auto-replied\r\nContent-Type: text/plain\r\n\r\nI am on holiday\r\n")
BOUNCE = ("From: MAILER-DAEMON@evil.example\r\nTo: probe@sl.local\r\nSubject: bounce\r\n"
          "Content-Type: multipart/report; report-type=delivery-status; boundary=\"b\"\r\n\r\n"
          "--b\r\nContent-Type: text/plain\r\n\r\nDelivery failed\r\n"
          "--b\r\nContent-Type: message/delivery-status\r\n\r\nStatus: 5.1.1\r\n--b--\r\n")
NONBOUNCE = ("From: attacker@evil.example\r\nTo: probe@sl.local\r\nSubject: hi\r\n"
             "Content-Type: text/plain\r\n\r\nhello\r\n")

def run_handle_data(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    env.original_content = raw.encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

def old_fwd(elid): return "%s%s%s" % (config.BOUNCE_PREFIX, elid, config.BOUNCE_SUFFIX)
def old_rep(elid): return "%s+%s+@%s" % (config.BOUNCE_PREFIX_FOR_REPLY_PHASE, elid, config.EMAIL_DOMAIN)

pgver = Session.execute("SELECT version()").scalar()
print("=== DB ENV FACTS ===")
print("DB_URI(host:port/db) = %s" % config.DB_URI.split("@")[-1])
print("SELECT version()     = %s" % pgver)
print("python               = %s" % sys.version.split()[0])
print("aiosmtpd             = %s" % aiosmtpd.__version__)
print("=== END ENV FACTS ===\n")

summary = []
transaction = connection.begin()
with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    try:
        INV = 99999999999999
        print("===== PART A: OUT-OF-OFFICE matrix via handle_DATA (Auto-Submitted:auto-replied) =====")
        uA = create_new_user(); aA = Alias.create_new_random(uA); Session.commit()
        cA = Contact.create(user_id=uA.id, alias_id=aA.id, website_email="ca@example.com", reply_email="repa@sl.local", commit=True)
        f_ooo = EmailLog.create(user_id=uA.id, contact_id=cA.id, alias_id=aA.id, is_reply=False, commit=True).id
        r_ooo = EmailLog.create(user_id=uA.id, contact_id=cA.id, alias_id=aA.id, is_reply=True, commit=True).id
        print("FIXTURES(OOO) user.id=%s alias=%s fwd_id=%s reply_id=%s INVALID=%s"
              % (uA.id, aA.email, f_ooo, r_ooo, INV))
        ooo_cells = [
          ("FWD OLD    VALID   OOO", "postmaster@corp.example", old_fwd(f_ooo)),
          ("FWD OLD    INVALID OOO", "postmaster@corp.example", old_fwd(INV)),
          ("FWD SIGNED VALID   OOO", "postmaster@corp.example", generate_verp_email(VerpType.bounce_forward, f_ooo)),
          ("FWD SIGNED INVALID OOO", "postmaster@corp.example", generate_verp_email(VerpType.bounce_forward, INV)),
          ("REP OLD    VALID   OOO", "postmaster@corp.example", old_rep(r_ooo)),
          ("REP OLD    INVALID OOO", "postmaster@corp.example", old_rep(INV)),
          ("REP SIGNED VALID   OOO", "postmaster@corp.example", generate_verp_email(VerpType.bounce_reply, r_ooo)),
          ("REP SIGNED INVALID OOO", "postmaster@corp.example", generate_verp_email(VerpType.bounce_reply, INV)),
        ]
        for label, mf, rcpt in ooo_cells:
            r = run_handle_data(mf, rcpt, OOO)
            print("CELL %-24s rcpt=%r\n     handle_DATA() = %r" % (label, rcpt, r))
            summary.append((label, r))

        print("\n===== PART B: iCloud branch via handle_DATA (bounce addr in mail_from, alias in rcpt) =====")
        uB = create_new_user(); aB = Alias.create_new_random(uB); Session.commit()
        cB = Contact.create(user_id=uB.id, alias_id=aB.id, website_email="cb@example.com", reply_email="repb@sl.local", commit=True)
        elB = EmailLog.create(user_id=uB.id, contact_id=cB.id, alias_id=aB.id, is_reply=False, commit=True)
        aB_email = aB.email; elB_id = elB.id
        for label, elid in [("iCloud OLD VALID   mail_from", elB_id), ("iCloud OLD INVALID mail_from", INV)]:
            r = run_handle_data(old_fwd(elid), aB_email, NONBOUNCE)
            print("CELL %-28s mail_from=%r rcpt=%r\n     handle_DATA() = %r" % (label, old_fwd(elid), aB_email, r))
            summary.append((label, r))

        print("\n===== PART C: committed STATE TRANSITION via handle_DATA (forward genuine bounce) =====")
        uC = create_new_user(); aC = Alias.create_new_random(uC); Session.commit()
        cC = Contact.create(user_id=uC.id, alias_id=aC.id, website_email="cc@example.com", reply_email="repc@sl.local", commit=True)
        elC = EmailLog.create(user_id=uC.id, contact_id=cC.id, alias_id=aC.id, is_reply=False, commit=True)
        elC_id = elC.id
        b0 = Bounce.filter().count(); r0 = RefusedEmail.filter().count()
        before = EmailLog.get(elC_id)
        print("BEFORE : email_log(id=%s).bounced=%s refused_email_id=%s bounced_mailbox_id=%s | Bounce_rows=%s RefusedEmail_rows=%s"
              % (elC_id, before.bounced, before.refused_email_id, before.bounced_mailbox_id, b0, r0))
        rc = run_handle_data("<>", generate_verp_email(VerpType.bounce_forward, elC_id), BOUNCE)
        print("HANDLER: handle_DATA() returned %r" % rc)
        after = EmailLog.get(elC_id)
        b1 = Bounce.filter().count(); r1 = RefusedEmail.filter().count()
        print("AFTER  : email_log(id=%s).bounced=%s refused_email_id=%s bounced_mailbox_id=%s | Bounce_rows=%s RefusedEmail_rows=%s"
              % (elC_id, after.bounced, after.refused_email_id, after.bounced_mailbox_id, b1, r1))
        print("DELTA  : Bounce_rows %+d, RefusedEmail_rows %+d" % (b1 - b0, r1 - r0))
        summary.append(("STATE handle_DATA bounce", rc))
        summary.append(("STATE bounced->%s refused_id_set=%s mbox_id_set=%s Bounce%+d Refused%+d"
                        % (after.bounced, after.refused_email_id is not None, after.bounced_mailbox_id is not None, b1 - b0, r1 - r0), "committed"))
    finally:
        transaction.rollback(); Session.rollback(); Session.close()

print("\n===== GROUP-B OUTCOME SUMMARY (handle_DATA) =====")
for lbl, r in summary:
    print("%-52s => %s" % (lbl, r))
print("===== END SUMMARY (%d rows) =====" % len(summary))
print("OBS_GROUP_B_DONE")
```

**Complete verbatim transcript — RUN 1 (`gb_run1.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/otntpkszmeztdrxguenw
Upload files to local dir
>>> init logging <<<
2026-07-14 02:26:12,393 - SL - DEBUG - 19630 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-14 02:26:13,491 - SL - DEBUG - 19630 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-14 02:26:13,492 - SL - DEBUG - 19630 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-14 02:26:13,493 - SL - DEBUG - 19630 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 02:26:13,494 - SL - DEBUG - 19630 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
=== DB ENV FACTS ===
DB_URI(host:port/db) = localhost:15432/test
SELECT version()     = PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
python               = 3.10.18
aiosmtpd             = 1.4.2
=== END ENV FACTS ===

===== PART A: OUT-OF-OFFICE matrix via handle_DATA (Auto-Submitted:auto-replied) =====
2026-07-14 02:26:13,796 - SL - INFO - 19630 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:26:13,810 - SL - DEBUG - 19630 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email elixir_stoups062@sl.local
2026-07-14 02:26:13,817 - SL - INFO - 19630 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURES(OOO) user.id=456 alias=elixir_stoups062@sl.local fwd_id=349 reply_id=350 INVALID=99999999999999
2026-07-14 02:26:13,846 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() -  - set message_id 6a0b3853-8710-4172-b612-484b817b7770
2026-07-14 02:26:13,846 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - 6a0b3853-8710-4172-b612-484b817b7770 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:13,846 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - 6a0b3853-8710-4172-b612-484b817b7770 - New message, mail from postmaster@corp.example, rctp tos ['bounce+349+@sl.local']
2026-07-14 02:26:13,847 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - 6a0b3853-8710-4172-b612-484b817b7770 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:13,847 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - 6a0b3853-8710-4172-b612-484b817b7770 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:13,848 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - 6a0b3853-8710-4172-b612-484b817b7770 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['bounce+349+@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:13,853 - SL - DEBUG - 19630 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - 6a0b3853-8710-4172-b612-484b817b7770 - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:26:13,854 - SL - DEBUG - 19630 - "/work/email_handler.py:2264" - handle_out_of_office_forward_phase() - 6a0b3853-8710-4172-b612-484b817b7770 - send the out-of-office email to the contact <Contact 158 ca@example.com 765>, old to_header:probe@sl.local rcpt_tos:['bounce+349+@sl.local'] <EmailLog 349>
2026-07-14 02:26:13,854 - SL - DEBUG - 19630 - "/work/email_handler.py:2280" - handle_out_of_office_forward_phase() - 6a0b3853-8710-4172-b612-484b817b7770 - after out-of-office transformation to_header:['repa@sl.local'] reply_to:None rcpt_tos:['repa@sl.local']
2026-07-14 02:26:13,856 - SL - DEBUG - 19630 - "/work/email_handler.py:2196" - handle() - 6a0b3853-8710-4172-b612-484b817b7770 - Reply phase postmaster@corp.example(postmaster@corp.example) -> repa@sl.local
2026-07-14 02:26:13,858 - SL - INFO - 19630 - "/work/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 6a0b3853-8710-4172-b612-484b817b7770 - DMARC check disabled
2026-07-14 02:26:13,862 - SL - WARNING - 19630 - "/work/email_handler.py:1393" - handle_unknown_mailbox() - 6a0b3853-8710-4172-b612-484b817b7770 - Reply email can only be used by mailbox. Actual mail_from: postmaster@corp.example. msg from header: postmaster@corp.example, reverse-alias repa@sl.local, <Alias 765 elixir_stoups062@sl.local> <User 456 Test User user_w4w53r9tmb@mailbox.test> <Contact 158 ca@example.com 765>
2026-07-14 02:26:13,882 - SL - DEBUG - 19630 - "/work/app/email_utils.py:303" - send_email() - 6a0b3853-8710-4172-b612-484b817b7770 - send email to user_w4w53r9tmb@mailbox.test, subject 'Attempt to use your alias elixir_stoups062@sl.local from postmaster@corp.example'
2026-07-14 02:26:13,887 - SL - DEBUG - 19630 - "/work/app/mail_sender.py:131" - send() - 6a0b3853-8710-4172-b612-484b817b7770 - send email with subject 'Attempt to use your alias elixir_stoups062@sl.local from postmaster@corp.example', from '"noreply@sl.local" <noreply@sl.local>' to 'user_w4w53r9tmb@mailbox.test'
2026-07-14 02:26:13,887 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - 6a0b3853-8710-4172-b612-484b817b7770 - Finish mail_from postmaster@corp.example, rcpt_tos ['repa@sl.local'], takes 0.04110860824584961 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
CELL FWD OLD    VALID   OOO   rcpt='bounce+349+@sl.local'
     handle_DATA() = '250 SL E214 Unauthorized for using reverse alias'
2026-07-14 02:26:13,888 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - 6a0b3853-8710-4172-b612-484b817b7770 - set message_id 1357ec7c-086f-46e8-a6d0-821bbe881932
2026-07-14 02:26:13,888 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - 1357ec7c-086f-46e8-a6d0-821bbe881932 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:13,888 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - 1357ec7c-086f-46e8-a6d0-821bbe881932 - New message, mail from postmaster@corp.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-14 02:26:13,888 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - 1357ec7c-086f-46e8-a6d0-821bbe881932 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:13,888 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - 1357ec7c-086f-46e8-a6d0-821bbe881932 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:13,890 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - 1357ec7c-086f-46e8-a6d0-821bbe881932 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:13,893 - SL - WARNING - 19630 - "/work/email_handler.py:2066" - handle() - 1357ec7c-086f-46e8-a6d0-821bbe881932 - No such email log
2026-07-14 02:26:13,893 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - 1357ec7c-086f-46e8-a6d0-821bbe881932 - Finish mail_from postmaster@corp.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.005699872970581055 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD OLD    INVALID OOO   rcpt='bounce+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:26:13,894 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - 1357ec7c-086f-46e8-a6d0-821bbe881932 - set message_id e4ab69e8-3f79-45a9-b21e-d354a3f41c57
2026-07-14 02:26:13,894 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:13,894 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - New message, mail from postmaster@corp.example, rctp tos ['sl.lmycyibtgq4syibsgm4dgmzugzoq.gasefkay7hte6@sl.local']
2026-07-14 02:26:13,895 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:13,895 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:13,896 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['sl.lmycyibtgq4syibsgm4dgmzugzoq.gasefkay7hte6@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:13,900 - SL - DEBUG - 19630 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:26:13,901 - SL - DEBUG - 19630 - "/work/email_handler.py:2264" - handle_out_of_office_forward_phase() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - send the out-of-office email to the contact <Contact 158 ca@example.com 765>, old to_header:probe@sl.local rcpt_tos:['sl.lmycyibtgq4syibsgm4dgmzugzoq.gasefkay7hte6@sl.local'] <EmailLog 349>
2026-07-14 02:26:13,901 - SL - DEBUG - 19630 - "/work/email_handler.py:2280" - handle_out_of_office_forward_phase() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - after out-of-office transformation to_header:['repa@sl.local'] reply_to:None rcpt_tos:['repa@sl.local']
2026-07-14 02:26:13,902 - SL - DEBUG - 19630 - "/work/email_handler.py:2196" - handle() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - Reply phase postmaster@corp.example(postmaster@corp.example) -> repa@sl.local
2026-07-14 02:26:13,909 - SL - INFO - 19630 - "/work/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - DMARC check disabled
2026-07-14 02:26:13,910 - SL - WARNING - 19630 - "/work/email_handler.py:1393" - handle_unknown_mailbox() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - Reply email can only be used by mailbox. Actual mail_from: postmaster@corp.example. msg from header: postmaster@corp.example, reverse-alias repa@sl.local, <Alias 765 elixir_stoups062@sl.local> <User 456 Test User user_w4w53r9tmb@mailbox.test> <Contact 158 ca@example.com 765>
2026-07-14 02:26:13,928 - SL - DEBUG - 19630 - "/work/app/email_utils.py:303" - send_email() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - send email to user_w4w53r9tmb@mailbox.test, subject 'Attempt to use your alias elixir_stoups062@sl.local from postmaster@corp.example'
2026-07-14 02:26:13,932 - SL - DEBUG - 19630 - "/work/app/mail_sender.py:131" - send() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - send email with subject 'Attempt to use your alias elixir_stoups062@sl.local from postmaster@corp.example', from '"noreply@sl.local" <noreply@sl.local>' to 'user_w4w53r9tmb@mailbox.test'
2026-07-14 02:26:13,932 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - Finish mail_from postmaster@corp.example, rcpt_tos ['repa@sl.local'], takes 0.03781771659851074 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
CELL FWD SIGNED VALID   OOO   rcpt='sl.lmycyibtgq4syibsgm4dgmzugzoq.gasefkay7hte6@sl.local'
     handle_DATA() = '250 SL E214 Unauthorized for using reverse alias'
2026-07-14 02:26:13,933 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - e4ab69e8-3f79-45a9-b21e-d354a3f41c57 - set message_id 4055dab8-e93e-4976-8c4a-2b468b079178
2026-07-14 02:26:13,933 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - 4055dab8-e93e-4976-8c4a-2b468b079178 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:13,933 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - 4055dab8-e93e-4976-8c4a-2b468b079178 - New message, mail from postmaster@corp.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local']
2026-07-14 02:26:13,933 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - 4055dab8-e93e-4976-8c4a-2b468b079178 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:13,933 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - 4055dab8-e93e-4976-8c4a-2b468b079178 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:13,935 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - 4055dab8-e93e-4976-8c4a-2b468b079178 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:13,938 - SL - WARNING - 19630 - "/work/email_handler.py:2066" - handle() - 4055dab8-e93e-4976-8c4a-2b468b079178 - No such email log
2026-07-14 02:26:13,938 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - 4055dab8-e93e-4976-8c4a-2b468b079178 - Finish mail_from postmaster@corp.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local'], takes 0.0058002471923828125 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD SIGNED INVALID OOO   rcpt='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:26:13,939 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - 4055dab8-e93e-4976-8c4a-2b468b079178 - set message_id 388340d7-237d-4a5e-b39f-f52d873e63b4
2026-07-14 02:26:13,939 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:13,939 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - New message, mail from postmaster@corp.example, rctp tos ['bounce_reply+350+@sl.local']
2026-07-14 02:26:13,940 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:13,940 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:13,941 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['bounce_reply+350+@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:13,944 - SL - DEBUG - 19630 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:26:13,947 - SL - DEBUG - 19630 - "/work/email_handler.py:2238" - handle_out_of_office_reply_phase() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - send the out-of-office email to the alias <Alias 765 elixir_stoups062@sl.local>, old to_header:probe@sl.local rcpt_tos:['bounce_reply+350+@sl.local'], <EmailLog 350>
2026-07-14 02:26:13,947 - SL - DEBUG - 19630 - "/work/email_handler.py:2254" - handle_out_of_office_reply_phase() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - after out-of-office transformation to_header:['elixir_stoups062@sl.local'] reply_to:None rcpt_tos:['elixir_stoups062@sl.local']
2026-07-14 02:26:13,949 - SL - DEBUG - 19630 - "/work/email_handler.py:2202" - handle() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Forward phase postmaster@corp.example(postmaster@corp.example) -> elixir_stoups062@sl.local
2026-07-14 02:26:13,957 - SL - DEBUG - 19630 - "/work/email_handler.py:580" - handle_forward() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Create or get contact for from_header:postmaster@corp.example
2026-07-14 02:26:13,978 - SL - DEBUG - 19630 - "/work/app/contact_utils.py:110" - create_contact() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Created contact <Contact 159 postmaster@corp.example 765> for alias <Alias 765 elixir_stoups062@sl.local> with email postmaster@corp.example invalid_email=False
2026-07-14 02:26:13,978 - SL - INFO - 19630 - "/work/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - DMARC check disabled
2026-07-14 02:26:13,985 - SL - DEBUG - 19630 - "/work/email_handler.py:688" - forward_email_to_mailbox() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Forward <Contact 159 postmaster@corp.example 765> -> <Alias 765 elixir_stoups062@sl.local> -> <Mailbox 532 user_w4w53r9tmb@mailbox.test>
2026-07-14 02:26:13,988 - SL - DEBUG - 19630 - "/work/email_handler.py:740" - forward_email_to_mailbox() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Create <EmailLog 351> for <Contact 159 postmaster@corp.example 765>, <User 456 Test User user_w4w53r9tmb@mailbox.test>, <Mailbox 532 user_w4w53r9tmb@mailbox.test>
2026-07-14 02:26:13,992 - SL - WARNING - 19630 - "/work/email_handler.py:857" - forward_email_to_mailbox() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - missing date header, create one
2026-07-14 02:26:13,993 - SL - DEBUG - 19630 - "/work/email_handler.py:867" - forward_email_to_mailbox() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - From header, new:"postmaster at corp.example" <postmaster_at_corp_example_nmfgfxm@sl.local>, old:postmaster@corp.example
2026-07-14 02:26:13,993 - SL - DEBUG - 19630 - "/work/email_handler.py:316" - replace_header_when_forward() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Delete Cc header, old value None
2026-07-14 02:26:13,993 - SL - DEBUG - 19630 - "/work/email_handler.py:313" - replace_header_when_forward() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Replace To header, old: elixir_stoups062@sl.local, new: elixir_stoups062@sl.local
2026-07-14 02:26:13,993 - SL - INFO - 19630 - "/work/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Email has no unsubscribe header
2026-07-14 02:26:13,995 - SL - DEBUG - 19630 - "/work/email_handler.py:893" - forward_email_to_mailbox() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Forward mail from postmaster@corp.example to user_w4w53r9tmb@mailbox.test, mail_options:[], rcpt_options:[]
2026-07-14 02:26:13,995 - SL - DEBUG - 19630 - "/work/app/mail_sender.py:131" - send() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - send email with subject 'Away', from '"postmaster at corp.example" <postmaster_at_corp_example_nmfgfxm@sl.local>' to 'elixir_stoups062@sl.local'
2026-07-14 02:26:13,996 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - Finish mail_from postmaster@corp.example, rcpt_tos ['elixir_stoups062@sl.local'], takes 0.0565643310546875 seconds with return code '250 Message accepted for delivery'<<===
CELL REP OLD    VALID   OOO   rcpt='bounce_reply+350+@sl.local'
     handle_DATA() = '250 Message accepted for delivery'
2026-07-14 02:26:13,996 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - 388340d7-237d-4a5e-b39f-f52d873e63b4 - set message_id 4a883c1d-fb47-4d81-a717-eaca4436a360
2026-07-14 02:26:13,996 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - 4a883c1d-fb47-4d81-a717-eaca4436a360 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:13,996 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - 4a883c1d-fb47-4d81-a717-eaca4436a360 - New message, mail from postmaster@corp.example, rctp tos ['bounce_reply+99999999999999+@sl.local']
2026-07-14 02:26:13,997 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - 4a883c1d-fb47-4d81-a717-eaca4436a360 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:13,997 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - 4a883c1d-fb47-4d81-a717-eaca4436a360 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:13,998 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - 4a883c1d-fb47-4d81-a717-eaca4436a360 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['bounce_reply+99999999999999+@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:14,001 - SL - WARNING - 19630 - "/work/email_handler.py:2086" - handle() - 4a883c1d-fb47-4d81-a717-eaca4436a360 - No such email log
2026-07-14 02:26:14,001 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - 4a883c1d-fb47-4d81-a717-eaca4436a360 - Finish mail_from postmaster@corp.example, rcpt_tos ['bounce_reply+99999999999999+@sl.local'], takes 0.005011796951293945 seconds with return code '550 SL E512 No such email log'<<===
CELL REP OLD    INVALID OOO   rcpt='bounce_reply+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:26:14,002 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - 4a883c1d-fb47-4d81-a717-eaca4436a360 - set message_id fd47d640-8f0c-4790-8b6b-a8248ba65100
2026-07-14 02:26:14,002 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:14,002 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - New message, mail from postmaster@corp.example, rctp tos ['sl.lmysyibtguycyibsgm4dgmzugzoq.awmia3ereye3u@sl.local']
2026-07-14 02:26:14,003 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:14,003 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:14,004 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['sl.lmysyibtguycyibsgm4dgmzugzoq.awmia3ereye3u@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:14,007 - SL - DEBUG - 19630 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:26:14,008 - SL - DEBUG - 19630 - "/work/email_handler.py:2238" - handle_out_of_office_reply_phase() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - send the out-of-office email to the alias <Alias 765 elixir_stoups062@sl.local>, old to_header:probe@sl.local rcpt_tos:['sl.lmysyibtguycyibsgm4dgmzugzoq.awmia3ereye3u@sl.local'], <EmailLog 350>
2026-07-14 02:26:14,008 - SL - DEBUG - 19630 - "/work/email_handler.py:2254" - handle_out_of_office_reply_phase() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - after out-of-office transformation to_header:['elixir_stoups062@sl.local'] reply_to:None rcpt_tos:['elixir_stoups062@sl.local']
2026-07-14 02:26:14,010 - SL - DEBUG - 19630 - "/work/email_handler.py:2202" - handle() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Forward phase postmaster@corp.example(postmaster@corp.example) -> elixir_stoups062@sl.local
2026-07-14 02:26:14,017 - SL - DEBUG - 19630 - "/work/email_handler.py:580" - handle_forward() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Create or get contact for from_header:postmaster@corp.example
2026-07-14 02:26:14,018 - SL - INFO - 19630 - "/work/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - DMARC check disabled
2026-07-14 02:26:14,024 - SL - DEBUG - 19630 - "/work/email_handler.py:688" - forward_email_to_mailbox() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Forward <Contact 159 postmaster@corp.example 765> -> <Alias 765 elixir_stoups062@sl.local> -> <Mailbox 532 user_w4w53r9tmb@mailbox.test>
2026-07-14 02:26:14,026 - SL - DEBUG - 19630 - "/work/email_handler.py:740" - forward_email_to_mailbox() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Create <EmailLog 352> for <Contact 159 postmaster@corp.example 765>, <User 456 Test User user_w4w53r9tmb@mailbox.test>, <Mailbox 532 user_w4w53r9tmb@mailbox.test>
2026-07-14 02:26:14,031 - SL - WARNING - 19630 - "/work/email_handler.py:857" - forward_email_to_mailbox() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - missing date header, create one
2026-07-14 02:26:14,031 - SL - DEBUG - 19630 - "/work/email_handler.py:867" - forward_email_to_mailbox() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - From header, new:"postmaster at corp.example" <postmaster_at_corp_example_nmfgfxm@sl.local>, old:postmaster@corp.example
2026-07-14 02:26:14,031 - SL - DEBUG - 19630 - "/work/email_handler.py:316" - replace_header_when_forward() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Delete Cc header, old value None
2026-07-14 02:26:14,031 - SL - DEBUG - 19630 - "/work/email_handler.py:313" - replace_header_when_forward() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Replace To header, old: elixir_stoups062@sl.local, new: elixir_stoups062@sl.local
2026-07-14 02:26:14,031 - SL - INFO - 19630 - "/work/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Email has no unsubscribe header
2026-07-14 02:26:14,034 - SL - DEBUG - 19630 - "/work/email_handler.py:893" - forward_email_to_mailbox() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Forward mail from postmaster@corp.example to user_w4w53r9tmb@mailbox.test, mail_options:[], rcpt_options:[]
2026-07-14 02:26:14,034 - SL - DEBUG - 19630 - "/work/app/mail_sender.py:131" - send() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - send email with subject 'Away', from '"postmaster at corp.example" <postmaster_at_corp_example_nmfgfxm@sl.local>' to 'elixir_stoups062@sl.local'
2026-07-14 02:26:14,034 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - Finish mail_from postmaster@corp.example, rcpt_tos ['elixir_stoups062@sl.local'], takes 0.03189206123352051 seconds with return code '250 Message accepted for delivery'<<===
CELL REP SIGNED VALID   OOO   rcpt='sl.lmysyibtguycyibsgm4dgmzugzoq.awmia3ereye3u@sl.local'
     handle_DATA() = '250 Message accepted for delivery'
2026-07-14 02:26:14,035 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - fd47d640-8f0c-4790-8b6b-a8248ba65100 - set message_id 1ba434e5-4241-4589-a657-9b324c6c8149
2026-07-14 02:26:14,035 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - 1ba434e5-4241-4589-a657-9b324c6c8149 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:14,035 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - 1ba434e5-4241-4589-a657-9b324c6c8149 - New message, mail from postmaster@corp.example, rctp tos ['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local']
2026-07-14 02:26:14,035 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - 1ba434e5-4241-4589-a657-9b324c6c8149 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:14,035 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - 1ba434e5-4241-4589-a657-9b324c6c8149 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:14,036 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - 1ba434e5-4241-4589-a657-9b324c6c8149 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:14,040 - SL - WARNING - 19630 - "/work/email_handler.py:2086" - handle() - 1ba434e5-4241-4589-a657-9b324c6c8149 - No such email log
2026-07-14 02:26:14,040 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - 1ba434e5-4241-4589-a657-9b324c6c8149 - Finish mail_from postmaster@corp.example, rcpt_tos ['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local'], takes 0.004985332489013672 seconds with return code '550 SL E512 No such email log'<<===
CELL REP SIGNED INVALID OOO   rcpt='sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local'
     handle_DATA() = '550 SL E512 No such email log'

===== PART B: iCloud branch via handle_DATA (bounce addr in mail_from, alias in rcpt) =====
2026-07-14 02:26:14,297 - SL - INFO - 19630 - "/work/app/events/event_dispatcher.py:62" - send_event() - 1ba434e5-4241-4589-a657-9b324c6c8149 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:26:14,308 - SL - DEBUG - 19630 - "/work/app/models.py:1459" - generate_random_alias_email() - 1ba434e5-4241-4589-a657-9b324c6c8149 - generate email ramify_hoopla407@sl.local
2026-07-14 02:26:14,316 - SL - INFO - 19630 - "/work/app/events/event_dispatcher.py:62" - send_event() - 1ba434e5-4241-4589-a657-9b324c6c8149 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:26:14,331 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - 1ba434e5-4241-4589-a657-9b324c6c8149 - set message_id 428210c2-006d-4fb6-a568-50c05267c8af
2026-07-14 02:26:14,332 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - 428210c2-006d-4fb6-a568-50c05267c8af - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:14,332 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - 428210c2-006d-4fb6-a568-50c05267c8af - New message, mail from bounce+353+@sl.local, rctp tos ['ramify_hoopla407@sl.local']
2026-07-14 02:26:14,332 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - 428210c2-006d-4fb6-a568-50c05267c8af - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:14,332 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - 428210c2-006d-4fb6-a568-50c05267c8af - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:14,333 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - 428210c2-006d-4fb6-a568-50c05267c8af - ==>> Handle mail_from:bounce+353+@sl.local, rcpt_tos:['ramify_hoopla407@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:14,341 - SL - WARNING - 19630 - "/work/email_handler.py:2110" - handle() - 428210c2-006d-4fb6-a568-50c05267c8af - iCloud bounces <EmailLog 353> <Alias 767 ramify_hoopla407@sl.local>, saved to
2026-07-14 02:26:14,342 - SL - DEBUG - 19630 - "/work/email_handler.py:1862" - handle_bounce() - 428210c2-006d-4fb6-a568-50c05267c8af - handle bounce for <EmailLog 353>, phase=forward, contact=<Contact 160 cb@example.com 767>, alias=<Alias 767 ramify_hoopla407@sl.local>
2026-07-14 02:26:14,344 - SL - ERROR - 19630 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - 428210c2-006d-4fb6-a568-50c05267c8af - Use <Alias 767 ramify_hoopla407@sl.local> default mailbox <Mailbox 533 user_wj67csztmq@mailbox.test>
NoneType: None
2026-07-14 02:26:14,344 - SL - WARNING - 19630 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - 428210c2-006d-4fb6-a568-50c05267c8af - cannot get bounce info, debug at
2026-07-14 02:26:14,346 - SL - DEBUG - 19630 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - 428210c2-006d-4fb6-a568-50c05267c8af - Handle forward bounce <Contact 160 cb@example.com 767> -> <Alias 767 ramify_hoopla407@sl.local> -> <Mailbox 533 user_wj67csztmq@mailbox.test>. <EmailLog 353>
2026-07-14 02:26:14,350 - SL - WARNING - 19630 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - 428210c2-006d-4fb6-a568-50c05267c8af - Cannot parse original message from bounce message <Alias 767 ramify_hoopla407@sl.local> <User 457 Test User user_wj67csztmq@mailbox.test> <Contact 160 cb@example.com 767> refused-emails/full-05166cbb-2409-470c-b560-d071e37dea0e.eml
2026-07-14 02:26:14,353 - SL - DEBUG - 19630 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - 428210c2-006d-4fb6-a568-50c05267c8af - Create refused email <Refused Email 117 None 2026-07-21T02:26:14.353018+00:00>
2026-07-14 02:26:14,363 - SL - DEBUG - 19630 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - 428210c2-006d-4fb6-a568-50c05267c8af - Inform user <User 457 Test User user_wj67csztmq@mailbox.test> about a bounce from contact <Contact 160 cb@example.com 767> to alias <Alias 767 ramify_hoopla407@sl.local>
2026-07-14 02:26:14,391 - SL - DEBUG - 19630 - "/work/app/email_utils.py:303" - send_email() - 428210c2-006d-4fb6-a568-50c05267c8af - send email to user_wj67csztmq@mailbox.test, subject 'An email sent to ramify_hoopla407@sl.local cannot be delivered to your mailbox'
2026-07-14 02:26:14,396 - SL - DEBUG - 19630 - "/work/app/mail_sender.py:131" - send() - 428210c2-006d-4fb6-a568-50c05267c8af - send email with subject 'An email sent to ramify_hoopla407@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_wj67csztmq@mailbox.test'
2026-07-14 02:26:14,396 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - 428210c2-006d-4fb6-a568-50c05267c8af - Finish mail_from bounce+353+@sl.local, rcpt_tos ['ramify_hoopla407@sl.local'], takes 0.06434178352355957 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
CELL iCloud OLD VALID   mail_from mail_from='bounce+353+@sl.local' rcpt='ramify_hoopla407@sl.local'
     handle_DATA() = '250 SL E211 Bounce Forward phase handled'
2026-07-14 02:26:14,397 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - 428210c2-006d-4fb6-a568-50c05267c8af - set message_id 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03
2026-07-14 02:26:14,397 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:14,397 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - New message, mail from bounce+99999999999999+@sl.local, rctp tos ['ramify_hoopla407@sl.local']
2026-07-14 02:26:14,397 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:14,397 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:14,399 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - ==>> Handle mail_from:bounce+99999999999999+@sl.local, rcpt_tos:['ramify_hoopla407@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:14,408 - SL - WARNING - 19630 - "/work/email_handler.py:2110" - handle() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - iCloud bounces None <Alias 767 ramify_hoopla407@sl.local>, saved to
2026-07-14 02:26:14,408 - SL - WARNING - 19630 - "/work/email_handler.py:1857" - handle_bounce() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - No such email log
2026-07-14 02:26:14,408 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - Finish mail_from bounce+99999999999999+@sl.local, rcpt_tos ['ramify_hoopla407@sl.local'], takes 0.011489152908325195 seconds with return code '550 SL E512 No such email log'<<===
CELL iCloud OLD INVALID mail_from mail_from='bounce+99999999999999+@sl.local' rcpt='ramify_hoopla407@sl.local'
     handle_DATA() = '550 SL E512 No such email log'

===== PART C: committed STATE TRANSITION via handle_DATA (forward genuine bounce) =====
2026-07-14 02:26:14,669 - SL - INFO - 19630 - "/work/app/events/event_dispatcher.py:62" - send_event() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:26:14,680 - SL - DEBUG - 19630 - "/work/app/models.py:1459" - generate_random_alias_email() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - generate email looped_payoff112@sl.local
2026-07-14 02:26:14,688 - SL - INFO - 19630 - "/work/app/events/event_dispatcher.py:62" - send_event() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - Not sending events because webhook is not configured and allowed to be empty
BEFORE : email_log(id=354).bounced=False refused_email_id=None bounced_mailbox_id=None | Bounce_rows=113 RefusedEmail_rows=114
2026-07-14 02:26:14,705 - SL - DEBUG - 19630 - "/work/app/log.py:24" - set_message_id() - 1e6c2ad0-3ff7-44cc-b79d-a38bbc04ce03 - set message_id a8bd88e0-62f3-4ea5-be73-bf665a12ab4b
2026-07-14 02:26:14,705 - SL - DEBUG - 19630 - "/work/email_handler.py:2342" - _handle() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:14,705 - SL - INFO - 19630 - "/work/email_handler.py:2343" - _handle() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - New message, mail from <>, rctp tos ['sl.lmycyibtgu2cyibsgm4dgmzugzoq.ua2fnxs2vudyo@sl.local']
2026-07-14 02:26:14,706 - SL - INFO - 19630 - "/work/email_handler.py:1956" - handle() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:14,706 - SL - DEBUG - 19630 - "/work/email_handler.py:1963" - handle() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:14,706 - SL - DEBUG - 19630 - "/work/email_handler.py:1980" - handle() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibtgu2cyibsgm4dgmzugzoq.ua2fnxs2vudyo@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:14,710 - SL - DEBUG - 19630 - "/work/email_handler.py:1862" - handle_bounce() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - handle bounce for <EmailLog 354>, phase=forward, contact=<Contact 161 cc@example.com 769>, alias=<Alias 769 looped_payoff112@sl.local>
2026-07-14 02:26:14,712 - SL - ERROR - 19630 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - Use <Alias 769 looped_payoff112@sl.local> default mailbox <Mailbox 534 user_f5gh3m2pqb@mailbox.test>
NoneType: None
2026-07-14 02:26:14,712 - SL - WARNING - 19630 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - cannot get bounce info, debug at
2026-07-14 02:26:14,713 - SL - DEBUG - 19630 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - Handle forward bounce <Contact 161 cc@example.com 769> -> <Alias 769 looped_payoff112@sl.local> -> <Mailbox 534 user_f5gh3m2pqb@mailbox.test>. <EmailLog 354>
2026-07-14 02:26:14,717 - SL - WARNING - 19630 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - Cannot parse original message from bounce message <Alias 769 looped_payoff112@sl.local> <User 458 Test User user_f5gh3m2pqb@mailbox.test> <Contact 161 cc@example.com 769> refused-emails/full-b49e72a7-7286-4d39-8494-4e8fbc0777e0.eml
2026-07-14 02:26:14,719 - SL - DEBUG - 19630 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - Create refused email <Refused Email 118 None 2026-07-21T02:26:14.719424+00:00>
2026-07-14 02:26:14,727 - SL - DEBUG - 19630 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - Inform user <User 458 Test User user_f5gh3m2pqb@mailbox.test> about a bounce from contact <Contact 161 cc@example.com 769> to alias <Alias 769 looped_payoff112@sl.local>
2026-07-14 02:26:14,751 - SL - DEBUG - 19630 - "/work/app/email_utils.py:303" - send_email() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - send email to user_f5gh3m2pqb@mailbox.test, subject 'An email sent to looped_payoff112@sl.local cannot be delivered to your mailbox'
2026-07-14 02:26:14,756 - SL - DEBUG - 19630 - "/work/app/mail_sender.py:131" - send() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - send email with subject 'An email sent to looped_payoff112@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_f5gh3m2pqb@mailbox.test'
2026-07-14 02:26:14,756 - SL - INFO - 19630 - "/work/email_handler.py:2367" - _handle() - a8bd88e0-62f3-4ea5-be73-bf665a12ab4b - Finish mail_from <>, rcpt_tos ['sl.lmycyibtgu2cyibsgm4dgmzugzoq.ua2fnxs2vudyo@sl.local'], takes 0.05091238021850586 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
HANDLER: handle_DATA() returned '250 SL E211 Bounce Forward phase handled'
AFTER  : email_log(id=354).bounced=True refused_email_id=118 bounced_mailbox_id=534 | Bounce_rows=114 RefusedEmail_rows=115
DELTA  : Bounce_rows +1, RefusedEmail_rows +1

===== GROUP-B OUTCOME SUMMARY (handle_DATA) =====
FWD OLD    VALID   OOO                               => 250 SL E214 Unauthorized for using reverse alias
FWD OLD    INVALID OOO                               => 550 SL E512 No such email log
FWD SIGNED VALID   OOO                               => 250 SL E214 Unauthorized for using reverse alias
FWD SIGNED INVALID OOO                               => 550 SL E512 No such email log
REP OLD    VALID   OOO                               => 250 Message accepted for delivery
REP OLD    INVALID OOO                               => 550 SL E512 No such email log
REP SIGNED VALID   OOO                               => 250 Message accepted for delivery
REP SIGNED INVALID OOO                               => 550 SL E512 No such email log
iCloud OLD VALID   mail_from                         => 250 SL E211 Bounce Forward phase handled
iCloud OLD INVALID mail_from                         => 550 SL E512 No such email log
STATE handle_DATA bounce                             => 250 SL E211 Bounce Forward phase handled
STATE bounced->True refused_id_set=True mbox_id_set=True Bounce+1 Refused+1 => committed
===== END SUMMARY (12 rows) =====
OBS_GROUP_B_DONE
```

**Complete verbatim transcript — RUN 2 (`gb_run2.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/adcpkapvyxcfezeysbbq
Upload files to local dir
>>> init logging <<<
2026-07-14 02:26:51,574 - SL - DEBUG - 19695 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-14 02:26:52,739 - SL - DEBUG - 19695 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-14 02:26:52,740 - SL - DEBUG - 19695 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-14 02:26:52,741 - SL - DEBUG - 19695 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 02:26:52,742 - SL - DEBUG - 19695 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
=== DB ENV FACTS ===
DB_URI(host:port/db) = localhost:15432/test
SELECT version()     = PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
python               = 3.10.18
aiosmtpd             = 1.4.2
=== END ENV FACTS ===

===== PART A: OUT-OF-OFFICE matrix via handle_DATA (Auto-Submitted:auto-replied) =====
2026-07-14 02:26:53,053 - SL - INFO - 19695 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:26:53,069 - SL - DEBUG - 19695 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email refill_niches366@sl.local
2026-07-14 02:26:53,078 - SL - INFO - 19695 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIXTURES(OOO) user.id=459 alias=refill_niches366@sl.local fwd_id=355 reply_id=356 INVALID=99999999999999
2026-07-14 02:26:53,109 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() -  - set message_id 05414da1-2036-42fd-a910-462cf356ac88
2026-07-14 02:26:53,109 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - 05414da1-2036-42fd-a910-462cf356ac88 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,109 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - 05414da1-2036-42fd-a910-462cf356ac88 - New message, mail from postmaster@corp.example, rctp tos ['bounce+355+@sl.local']
2026-07-14 02:26:53,110 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - 05414da1-2036-42fd-a910-462cf356ac88 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,110 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - 05414da1-2036-42fd-a910-462cf356ac88 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,112 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - 05414da1-2036-42fd-a910-462cf356ac88 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['bounce+355+@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,117 - SL - DEBUG - 19695 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - 05414da1-2036-42fd-a910-462cf356ac88 - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:26:53,118 - SL - DEBUG - 19695 - "/work/email_handler.py:2264" - handle_out_of_office_forward_phase() - 05414da1-2036-42fd-a910-462cf356ac88 - send the out-of-office email to the contact <Contact 162 ca@example.com 771>, old to_header:probe@sl.local rcpt_tos:['bounce+355+@sl.local'] <EmailLog 355>
2026-07-14 02:26:53,118 - SL - DEBUG - 19695 - "/work/email_handler.py:2280" - handle_out_of_office_forward_phase() - 05414da1-2036-42fd-a910-462cf356ac88 - after out-of-office transformation to_header:['repa@sl.local'] reply_to:None rcpt_tos:['repa@sl.local']
2026-07-14 02:26:53,120 - SL - DEBUG - 19695 - "/work/email_handler.py:2196" - handle() - 05414da1-2036-42fd-a910-462cf356ac88 - Reply phase postmaster@corp.example(postmaster@corp.example) -> repa@sl.local
2026-07-14 02:26:53,129 - SL - INFO - 19695 - "/work/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 05414da1-2036-42fd-a910-462cf356ac88 - DMARC check disabled
2026-07-14 02:26:53,130 - SL - WARNING - 19695 - "/work/email_handler.py:1393" - handle_unknown_mailbox() - 05414da1-2036-42fd-a910-462cf356ac88 - Reply email can only be used by mailbox. Actual mail_from: postmaster@corp.example. msg from header: postmaster@corp.example, reverse-alias repa@sl.local, <Alias 765 elixir_stoups062@sl.local> <User 456 Test User user_w4w53r9tmb@mailbox.test> <Contact 158 ca@example.com 765>
2026-07-14 02:26:53,150 - SL - DEBUG - 19695 - "/work/app/email_utils.py:303" - send_email() - 05414da1-2036-42fd-a910-462cf356ac88 - send email to user_w4w53r9tmb@mailbox.test, subject 'Attempt to use your alias elixir_stoups062@sl.local from postmaster@corp.example'
2026-07-14 02:26:53,155 - SL - DEBUG - 19695 - "/work/app/mail_sender.py:131" - send() - 05414da1-2036-42fd-a910-462cf356ac88 - send email with subject 'Attempt to use your alias elixir_stoups062@sl.local from postmaster@corp.example', from '"noreply@sl.local" <noreply@sl.local>' to 'user_w4w53r9tmb@mailbox.test'
2026-07-14 02:26:53,155 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - 05414da1-2036-42fd-a910-462cf356ac88 - Finish mail_from postmaster@corp.example, rcpt_tos ['repa@sl.local'], takes 0.04626178741455078 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
CELL FWD OLD    VALID   OOO   rcpt='bounce+355+@sl.local'
     handle_DATA() = '250 SL E214 Unauthorized for using reverse alias'
2026-07-14 02:26:53,156 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - 05414da1-2036-42fd-a910-462cf356ac88 - set message_id 29753b37-0074-4d82-8475-961de17bcf70
2026-07-14 02:26:53,156 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - 29753b37-0074-4d82-8475-961de17bcf70 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,156 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - 29753b37-0074-4d82-8475-961de17bcf70 - New message, mail from postmaster@corp.example, rctp tos ['bounce+99999999999999+@sl.local']
2026-07-14 02:26:53,157 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - 29753b37-0074-4d82-8475-961de17bcf70 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,157 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - 29753b37-0074-4d82-8475-961de17bcf70 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,158 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - 29753b37-0074-4d82-8475-961de17bcf70 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['bounce+99999999999999+@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,161 - SL - WARNING - 19695 - "/work/email_handler.py:2066" - handle() - 29753b37-0074-4d82-8475-961de17bcf70 - No such email log
2026-07-14 02:26:53,161 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - 29753b37-0074-4d82-8475-961de17bcf70 - Finish mail_from postmaster@corp.example, rcpt_tos ['bounce+99999999999999+@sl.local'], takes 0.005248308181762695 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD OLD    INVALID OOO   rcpt='bounce+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:26:53,162 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - 29753b37-0074-4d82-8475-961de17bcf70 - set message_id 3af0430d-8094-477f-bb2f-0628c37b3efe
2026-07-14 02:26:53,162 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - 3af0430d-8094-477f-bb2f-0628c37b3efe - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,162 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - 3af0430d-8094-477f-bb2f-0628c37b3efe - New message, mail from postmaster@corp.example, rctp tos ['sl.lmycyibtgu2syibsgm4dgmzugzoq.gj3wifa4fxlga@sl.local']
2026-07-14 02:26:53,162 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - 3af0430d-8094-477f-bb2f-0628c37b3efe - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,163 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - 3af0430d-8094-477f-bb2f-0628c37b3efe - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,163 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - 3af0430d-8094-477f-bb2f-0628c37b3efe - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['sl.lmycyibtgu2syibsgm4dgmzugzoq.gj3wifa4fxlga@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,167 - SL - DEBUG - 19695 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - 3af0430d-8094-477f-bb2f-0628c37b3efe - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:26:53,168 - SL - DEBUG - 19695 - "/work/email_handler.py:2264" - handle_out_of_office_forward_phase() - 3af0430d-8094-477f-bb2f-0628c37b3efe - send the out-of-office email to the contact <Contact 162 ca@example.com 771>, old to_header:probe@sl.local rcpt_tos:['sl.lmycyibtgu2syibsgm4dgmzugzoq.gj3wifa4fxlga@sl.local'] <EmailLog 355>
2026-07-14 02:26:53,168 - SL - DEBUG - 19695 - "/work/email_handler.py:2280" - handle_out_of_office_forward_phase() - 3af0430d-8094-477f-bb2f-0628c37b3efe - after out-of-office transformation to_header:['repa@sl.local'] reply_to:None rcpt_tos:['repa@sl.local']
2026-07-14 02:26:53,169 - SL - DEBUG - 19695 - "/work/email_handler.py:2196" - handle() - 3af0430d-8094-477f-bb2f-0628c37b3efe - Reply phase postmaster@corp.example(postmaster@corp.example) -> repa@sl.local
2026-07-14 02:26:53,173 - SL - INFO - 19695 - "/work/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 3af0430d-8094-477f-bb2f-0628c37b3efe - DMARC check disabled
2026-07-14 02:26:53,173 - SL - WARNING - 19695 - "/work/email_handler.py:1393" - handle_unknown_mailbox() - 3af0430d-8094-477f-bb2f-0628c37b3efe - Reply email can only be used by mailbox. Actual mail_from: postmaster@corp.example. msg from header: postmaster@corp.example, reverse-alias repa@sl.local, <Alias 765 elixir_stoups062@sl.local> <User 456 Test User user_w4w53r9tmb@mailbox.test> <Contact 158 ca@example.com 765>
2026-07-14 02:26:53,191 - SL - DEBUG - 19695 - "/work/app/email_utils.py:303" - send_email() - 3af0430d-8094-477f-bb2f-0628c37b3efe - send email to user_w4w53r9tmb@mailbox.test, subject 'Attempt to use your alias elixir_stoups062@sl.local from postmaster@corp.example'
2026-07-14 02:26:53,195 - SL - DEBUG - 19695 - "/work/app/mail_sender.py:131" - send() - 3af0430d-8094-477f-bb2f-0628c37b3efe - send email with subject 'Attempt to use your alias elixir_stoups062@sl.local from postmaster@corp.example', from '"noreply@sl.local" <noreply@sl.local>' to 'user_w4w53r9tmb@mailbox.test'
2026-07-14 02:26:53,195 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - 3af0430d-8094-477f-bb2f-0628c37b3efe - Finish mail_from postmaster@corp.example, rcpt_tos ['repa@sl.local'], takes 0.03312993049621582 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
CELL FWD SIGNED VALID   OOO   rcpt='sl.lmycyibtgu2syibsgm4dgmzugzoq.gj3wifa4fxlga@sl.local'
     handle_DATA() = '250 SL E214 Unauthorized for using reverse alias'
2026-07-14 02:26:53,196 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - 3af0430d-8094-477f-bb2f-0628c37b3efe - set message_id 440dca6d-22f6-4c99-92d8-0dab4bd74022
2026-07-14 02:26:53,196 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - 440dca6d-22f6-4c99-92d8-0dab4bd74022 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,196 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - 440dca6d-22f6-4c99-92d8-0dab4bd74022 - New message, mail from postmaster@corp.example, rctp tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local']
2026-07-14 02:26:53,196 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - 440dca6d-22f6-4c99-92d8-0dab4bd74022 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,196 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - 440dca6d-22f6-4c99-92d8-0dab4bd74022 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,197 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - 440dca6d-22f6-4c99-92d8-0dab4bd74022 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,201 - SL - WARNING - 19695 - "/work/email_handler.py:2066" - handle() - 440dca6d-22f6-4c99-92d8-0dab4bd74022 - No such email log
2026-07-14 02:26:53,201 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - 440dca6d-22f6-4c99-92d8-0dab4bd74022 - Finish mail_from postmaster@corp.example, rcpt_tos ['sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local'], takes 0.005177974700927734 seconds with return code '550 SL E512 No such email log'<<===
CELL FWD SIGNED INVALID OOO   rcpt='sl.lmycyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.oqmwglnwjcuku@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:26:53,201 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - 440dca6d-22f6-4c99-92d8-0dab4bd74022 - set message_id e3ffba75-61d5-4434-b6be-ce02474ecfaa
2026-07-14 02:26:53,201 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,201 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - New message, mail from postmaster@corp.example, rctp tos ['bounce_reply+356+@sl.local']
2026-07-14 02:26:53,202 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,202 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,203 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['bounce_reply+356+@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,206 - SL - DEBUG - 19695 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:26:53,209 - SL - DEBUG - 19695 - "/work/email_handler.py:2238" - handle_out_of_office_reply_phase() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - send the out-of-office email to the alias <Alias 771 refill_niches366@sl.local>, old to_header:probe@sl.local rcpt_tos:['bounce_reply+356+@sl.local'], <EmailLog 356>
2026-07-14 02:26:53,209 - SL - DEBUG - 19695 - "/work/email_handler.py:2254" - handle_out_of_office_reply_phase() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - after out-of-office transformation to_header:['refill_niches366@sl.local'] reply_to:None rcpt_tos:['refill_niches366@sl.local']
2026-07-14 02:26:53,211 - SL - DEBUG - 19695 - "/work/email_handler.py:2202" - handle() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Forward phase postmaster@corp.example(postmaster@corp.example) -> refill_niches366@sl.local
2026-07-14 02:26:53,218 - SL - DEBUG - 19695 - "/work/email_handler.py:580" - handle_forward() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Create or get contact for from_header:postmaster@corp.example
2026-07-14 02:26:53,240 - SL - DEBUG - 19695 - "/work/app/contact_utils.py:110" - create_contact() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Created contact <Contact 163 postmaster@corp.example 771> for alias <Alias 771 refill_niches366@sl.local> with email postmaster@corp.example invalid_email=False
2026-07-14 02:26:53,240 - SL - INFO - 19695 - "/work/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - DMARC check disabled
2026-07-14 02:26:53,250 - SL - DEBUG - 19695 - "/work/email_handler.py:688" - forward_email_to_mailbox() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Forward <Contact 163 postmaster@corp.example 771> -> <Alias 771 refill_niches366@sl.local> -> <Mailbox 535 user_mdgah17l3x@mailbox.test>
2026-07-14 02:26:53,253 - SL - DEBUG - 19695 - "/work/email_handler.py:740" - forward_email_to_mailbox() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Create <EmailLog 357> for <Contact 163 postmaster@corp.example 771>, <User 459 Test User user_mdgah17l3x@mailbox.test>, <Mailbox 535 user_mdgah17l3x@mailbox.test>
2026-07-14 02:26:53,258 - SL - WARNING - 19695 - "/work/email_handler.py:857" - forward_email_to_mailbox() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - missing date header, create one
2026-07-14 02:26:53,259 - SL - DEBUG - 19695 - "/work/email_handler.py:867" - forward_email_to_mailbox() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - From header, new:"postmaster at corp.example" <postmaster_at_corp_example_rgocd@sl.local>, old:postmaster@corp.example
2026-07-14 02:26:53,259 - SL - DEBUG - 19695 - "/work/email_handler.py:316" - replace_header_when_forward() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Delete Cc header, old value None
2026-07-14 02:26:53,259 - SL - DEBUG - 19695 - "/work/email_handler.py:313" - replace_header_when_forward() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Replace To header, old: refill_niches366@sl.local, new: refill_niches366@sl.local
2026-07-14 02:26:53,259 - SL - INFO - 19695 - "/work/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Email has no unsubscribe header
2026-07-14 02:26:53,261 - SL - DEBUG - 19695 - "/work/email_handler.py:893" - forward_email_to_mailbox() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Forward mail from postmaster@corp.example to user_mdgah17l3x@mailbox.test, mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,262 - SL - DEBUG - 19695 - "/work/app/mail_sender.py:131" - send() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - send email with subject 'Away', from '"postmaster at corp.example" <postmaster_at_corp_example_rgocd@sl.local>' to 'refill_niches366@sl.local'
2026-07-14 02:26:53,262 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - Finish mail_from postmaster@corp.example, rcpt_tos ['refill_niches366@sl.local'], takes 0.06059098243713379 seconds with return code '250 Message accepted for delivery'<<===
CELL REP OLD    VALID   OOO   rcpt='bounce_reply+356+@sl.local'
     handle_DATA() = '250 Message accepted for delivery'
2026-07-14 02:26:53,263 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - e3ffba75-61d5-4434-b6be-ce02474ecfaa - set message_id fd760067-fa4a-4f1f-bfa8-add3eba80f93
2026-07-14 02:26:53,263 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - fd760067-fa4a-4f1f-bfa8-add3eba80f93 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,263 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - fd760067-fa4a-4f1f-bfa8-add3eba80f93 - New message, mail from postmaster@corp.example, rctp tos ['bounce_reply+99999999999999+@sl.local']
2026-07-14 02:26:53,264 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - fd760067-fa4a-4f1f-bfa8-add3eba80f93 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,264 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - fd760067-fa4a-4f1f-bfa8-add3eba80f93 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,265 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - fd760067-fa4a-4f1f-bfa8-add3eba80f93 - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['bounce_reply+99999999999999+@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,269 - SL - WARNING - 19695 - "/work/email_handler.py:2086" - handle() - fd760067-fa4a-4f1f-bfa8-add3eba80f93 - No such email log
2026-07-14 02:26:53,269 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - fd760067-fa4a-4f1f-bfa8-add3eba80f93 - Finish mail_from postmaster@corp.example, rcpt_tos ['bounce_reply+99999999999999+@sl.local'], takes 0.005954742431640625 seconds with return code '550 SL E512 No such email log'<<===
CELL REP OLD    INVALID OOO   rcpt='bounce_reply+99999999999999+@sl.local'
     handle_DATA() = '550 SL E512 No such email log'
2026-07-14 02:26:53,270 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - fd760067-fa4a-4f1f-bfa8-add3eba80f93 - set message_id b56f91f7-8414-4b9a-a949-e3497ce8247e
2026-07-14 02:26:53,270 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - b56f91f7-8414-4b9a-a949-e3497ce8247e - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,270 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - b56f91f7-8414-4b9a-a949-e3497ce8247e - New message, mail from postmaster@corp.example, rctp tos ['sl.lmysyibtgu3cyibsgm4dgmzugzoq.b2w55gibhg6po@sl.local']
2026-07-14 02:26:53,270 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,271 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,272 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - b56f91f7-8414-4b9a-a949-e3497ce8247e - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['sl.lmysyibtgu3cyibsgm4dgmzugzoq.b2w55gibhg6po@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,275 - SL - DEBUG - 19695 - "/work/email_handler.py:1803" - is_automatic_out_of_office() - b56f91f7-8414-4b9a-a949-e3497ce8247e - out-of-office email Auto-Submitted:auto-replied
2026-07-14 02:26:53,276 - SL - DEBUG - 19695 - "/work/email_handler.py:2238" - handle_out_of_office_reply_phase() - b56f91f7-8414-4b9a-a949-e3497ce8247e - send the out-of-office email to the alias <Alias 771 refill_niches366@sl.local>, old to_header:probe@sl.local rcpt_tos:['sl.lmysyibtgu3cyibsgm4dgmzugzoq.b2w55gibhg6po@sl.local'], <EmailLog 356>
2026-07-14 02:26:53,277 - SL - DEBUG - 19695 - "/work/email_handler.py:2254" - handle_out_of_office_reply_phase() - b56f91f7-8414-4b9a-a949-e3497ce8247e - after out-of-office transformation to_header:['refill_niches366@sl.local'] reply_to:None rcpt_tos:['refill_niches366@sl.local']
2026-07-14 02:26:53,278 - SL - DEBUG - 19695 - "/work/email_handler.py:2202" - handle() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Forward phase postmaster@corp.example(postmaster@corp.example) -> refill_niches366@sl.local
2026-07-14 02:26:53,286 - SL - DEBUG - 19695 - "/work/email_handler.py:580" - handle_forward() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Create or get contact for from_header:postmaster@corp.example
2026-07-14 02:26:53,288 - SL - INFO - 19695 - "/work/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - b56f91f7-8414-4b9a-a949-e3497ce8247e - DMARC check disabled
2026-07-14 02:26:53,295 - SL - DEBUG - 19695 - "/work/email_handler.py:688" - forward_email_to_mailbox() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Forward <Contact 163 postmaster@corp.example 771> -> <Alias 771 refill_niches366@sl.local> -> <Mailbox 535 user_mdgah17l3x@mailbox.test>
2026-07-14 02:26:53,298 - SL - DEBUG - 19695 - "/work/email_handler.py:740" - forward_email_to_mailbox() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Create <EmailLog 358> for <Contact 163 postmaster@corp.example 771>, <User 459 Test User user_mdgah17l3x@mailbox.test>, <Mailbox 535 user_mdgah17l3x@mailbox.test>
2026-07-14 02:26:53,303 - SL - WARNING - 19695 - "/work/email_handler.py:857" - forward_email_to_mailbox() - b56f91f7-8414-4b9a-a949-e3497ce8247e - missing date header, create one
2026-07-14 02:26:53,303 - SL - DEBUG - 19695 - "/work/email_handler.py:867" - forward_email_to_mailbox() - b56f91f7-8414-4b9a-a949-e3497ce8247e - From header, new:"postmaster at corp.example" <postmaster_at_corp_example_rgocd@sl.local>, old:postmaster@corp.example
2026-07-14 02:26:53,303 - SL - DEBUG - 19695 - "/work/email_handler.py:316" - replace_header_when_forward() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Delete Cc header, old value None
2026-07-14 02:26:53,304 - SL - DEBUG - 19695 - "/work/email_handler.py:313" - replace_header_when_forward() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Replace To header, old: refill_niches366@sl.local, new: refill_niches366@sl.local
2026-07-14 02:26:53,304 - SL - INFO - 19695 - "/work/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Email has no unsubscribe header
2026-07-14 02:26:53,306 - SL - DEBUG - 19695 - "/work/email_handler.py:893" - forward_email_to_mailbox() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Forward mail from postmaster@corp.example to user_mdgah17l3x@mailbox.test, mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,306 - SL - DEBUG - 19695 - "/work/app/mail_sender.py:131" - send() - b56f91f7-8414-4b9a-a949-e3497ce8247e - send email with subject 'Away', from '"postmaster at corp.example" <postmaster_at_corp_example_rgocd@sl.local>' to 'refill_niches366@sl.local'
2026-07-14 02:26:53,306 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - b56f91f7-8414-4b9a-a949-e3497ce8247e - Finish mail_from postmaster@corp.example, rcpt_tos ['refill_niches366@sl.local'], takes 0.03681015968322754 seconds with return code '250 Message accepted for delivery'<<===
CELL REP SIGNED VALID   OOO   rcpt='sl.lmysyibtgu3cyibsgm4dgmzugzoq.b2w55gibhg6po@sl.local'
     handle_DATA() = '250 Message accepted for delivery'
2026-07-14 02:26:53,307 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - b56f91f7-8414-4b9a-a949-e3497ce8247e - set message_id b4af6de9-a07c-4415-a7c7-2854eb2f28aa
2026-07-14 02:26:53,307 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,307 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - New message, mail from postmaster@corp.example, rctp tos ['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local']
2026-07-14 02:26:53,308 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,308 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,309 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - ==>> Handle mail_from:postmaster@corp.example, rcpt_tos:['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local'], header_from:postmaster@corp.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'postmaster@corp.example'), ('To', 'probe@sl.local'), ('Subject', 'Away'), ('Auto-Submitted', 'auto-replied'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,314 - SL - WARNING - 19695 - "/work/email_handler.py:2086" - handle() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - No such email log
2026-07-14 02:26:53,314 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - Finish mail_from postmaster@corp.example, rcpt_tos ['sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local'], takes 0.00659632682800293 seconds with return code '550 SL E512 No such email log'<<===
CELL REP SIGNED INVALID OOO   rcpt='sl.lmysyibzhe4tsojzhe4tsojzhe4tslbagiztqmztgq3f2.4jxeahldf5voq@sl.local'
     handle_DATA() = '550 SL E512 No such email log'

===== PART B: iCloud branch via handle_DATA (bounce addr in mail_from, alias in rcpt) =====
2026-07-14 02:26:53,585 - SL - INFO - 19695 - "/work/app/events/event_dispatcher.py:62" - send_event() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:26:53,600 - SL - DEBUG - 19695 - "/work/app/models.py:1459" - generate_random_alias_email() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - generate email ripper_braise890@sl.local
2026-07-14 02:26:53,610 - SL - INFO - 19695 - "/work/app/events/event_dispatcher.py:62" - send_event() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:26:53,625 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - b4af6de9-a07c-4415-a7c7-2854eb2f28aa - set message_id c665ef3c-dafe-47e0-8e39-d404df738249
2026-07-14 02:26:53,625 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - c665ef3c-dafe-47e0-8e39-d404df738249 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,625 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - c665ef3c-dafe-47e0-8e39-d404df738249 - New message, mail from bounce+359+@sl.local, rctp tos ['ripper_braise890@sl.local']
2026-07-14 02:26:53,626 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - c665ef3c-dafe-47e0-8e39-d404df738249 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,626 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - c665ef3c-dafe-47e0-8e39-d404df738249 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,626 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - c665ef3c-dafe-47e0-8e39-d404df738249 - ==>> Handle mail_from:bounce+359+@sl.local, rcpt_tos:['ripper_braise890@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,634 - SL - WARNING - 19695 - "/work/email_handler.py:2110" - handle() - c665ef3c-dafe-47e0-8e39-d404df738249 - iCloud bounces <EmailLog 359> <Alias 773 ripper_braise890@sl.local>, saved to
2026-07-14 02:26:53,635 - SL - DEBUG - 19695 - "/work/email_handler.py:1862" - handle_bounce() - c665ef3c-dafe-47e0-8e39-d404df738249 - handle bounce for <EmailLog 359>, phase=forward, contact=<Contact 164 cb@example.com 773>, alias=<Alias 773 ripper_braise890@sl.local>
2026-07-14 02:26:53,637 - SL - ERROR - 19695 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - c665ef3c-dafe-47e0-8e39-d404df738249 - Use <Alias 773 ripper_braise890@sl.local> default mailbox <Mailbox 536 user_yejg1uhnbd@mailbox.test>
NoneType: None
2026-07-14 02:26:53,637 - SL - WARNING - 19695 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - c665ef3c-dafe-47e0-8e39-d404df738249 - cannot get bounce info, debug at
2026-07-14 02:26:53,639 - SL - DEBUG - 19695 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - c665ef3c-dafe-47e0-8e39-d404df738249 - Handle forward bounce <Contact 164 cb@example.com 773> -> <Alias 773 ripper_braise890@sl.local> -> <Mailbox 536 user_yejg1uhnbd@mailbox.test>. <EmailLog 359>
2026-07-14 02:26:53,643 - SL - WARNING - 19695 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - c665ef3c-dafe-47e0-8e39-d404df738249 - Cannot parse original message from bounce message <Alias 773 ripper_braise890@sl.local> <User 460 Test User user_yejg1uhnbd@mailbox.test> <Contact 164 cb@example.com 773> refused-emails/full-bbea167b-cdf1-42e8-8fe8-f941539e9c82.eml
2026-07-14 02:26:53,645 - SL - DEBUG - 19695 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - c665ef3c-dafe-47e0-8e39-d404df738249 - Create refused email <Refused Email 119 None 2026-07-21T02:26:53.645178+00:00>
2026-07-14 02:26:53,655 - SL - DEBUG - 19695 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - c665ef3c-dafe-47e0-8e39-d404df738249 - Inform user <User 460 Test User user_yejg1uhnbd@mailbox.test> about a bounce from contact <Contact 164 cb@example.com 773> to alias <Alias 773 ripper_braise890@sl.local>
2026-07-14 02:26:53,680 - SL - DEBUG - 19695 - "/work/app/email_utils.py:303" - send_email() - c665ef3c-dafe-47e0-8e39-d404df738249 - send email to user_yejg1uhnbd@mailbox.test, subject 'An email sent to ripper_braise890@sl.local cannot be delivered to your mailbox'
2026-07-14 02:26:53,685 - SL - DEBUG - 19695 - "/work/app/mail_sender.py:131" - send() - c665ef3c-dafe-47e0-8e39-d404df738249 - send email with subject 'An email sent to ripper_braise890@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_yejg1uhnbd@mailbox.test'
2026-07-14 02:26:53,685 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - c665ef3c-dafe-47e0-8e39-d404df738249 - Finish mail_from bounce+359+@sl.local, rcpt_tos ['ripper_braise890@sl.local'], takes 0.05997180938720703 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
CELL iCloud OLD VALID   mail_from mail_from='bounce+359+@sl.local' rcpt='ripper_braise890@sl.local'
     handle_DATA() = '250 SL E211 Bounce Forward phase handled'
2026-07-14 02:26:53,685 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - c665ef3c-dafe-47e0-8e39-d404df738249 - set message_id b96124a1-0636-4a1f-97ff-7c492cf29e45
2026-07-14 02:26:53,685 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:53,685 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - New message, mail from bounce+99999999999999+@sl.local, rctp tos ['ripper_braise890@sl.local']
2026-07-14 02:26:53,686 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:53,686 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:53,687 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - ==>> Handle mail_from:bounce+99999999999999+@sl.local, rcpt_tos:['ripper_braise890@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:53,698 - SL - WARNING - 19695 - "/work/email_handler.py:2110" - handle() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - iCloud bounces None <Alias 773 ripper_braise890@sl.local>, saved to
2026-07-14 02:26:53,698 - SL - WARNING - 19695 - "/work/email_handler.py:1857" - handle_bounce() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - No such email log
2026-07-14 02:26:53,698 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - Finish mail_from bounce+99999999999999+@sl.local, rcpt_tos ['ripper_braise890@sl.local'], takes 0.012454986572265625 seconds with return code '550 SL E512 No such email log'<<===
CELL iCloud OLD INVALID mail_from mail_from='bounce+99999999999999+@sl.local' rcpt='ripper_braise890@sl.local'
     handle_DATA() = '550 SL E512 No such email log'

===== PART C: committed STATE TRANSITION via handle_DATA (forward genuine bounce) =====
2026-07-14 02:26:53,965 - SL - INFO - 19695 - "/work/app/events/event_dispatcher.py:62" - send_event() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:26:53,980 - SL - DEBUG - 19695 - "/work/app/models.py:1459" - generate_random_alias_email() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - generate email casing_forage632@sl.local
2026-07-14 02:26:53,990 - SL - INFO - 19695 - "/work/app/events/event_dispatcher.py:62" - send_event() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - Not sending events because webhook is not configured and allowed to be empty
BEFORE : email_log(id=360).bounced=False refused_email_id=None bounced_mailbox_id=None | Bounce_rows=115 RefusedEmail_rows=116
2026-07-14 02:26:54,011 - SL - DEBUG - 19695 - "/work/app/log.py:24" - set_message_id() - b96124a1-0636-4a1f-97ff-7c492cf29e45 - set message_id 71778c78-3317-426d-9a8d-e7edf9e0bdcf
2026-07-14 02:26:54,012 - SL - DEBUG - 19695 - "/work/email_handler.py:2342" - _handle() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - ====>=====>====>====>====>====>====>====>
2026-07-14 02:26:54,012 - SL - INFO - 19695 - "/work/email_handler.py:2343" - _handle() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - New message, mail from <>, rctp tos ['sl.lmycyibtgyycyibsgm4dgmzugzoq.z3flh4lzme66k@sl.local']
2026-07-14 02:26:54,013 - SL - INFO - 19695 - "/work/email_handler.py:1956" - handle() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:26:54,013 - SL - DEBUG - 19695 - "/work/email_handler.py:1963" - handle() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - Cannot parse Postfix queue ID from None None
2026-07-14 02:26:54,014 - SL - DEBUG - 19695 - "/work/email_handler.py:1980" - handle() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - ==>> Handle mail_from:<>, rcpt_tos:['sl.lmycyibtgyycyibsgm4dgmzugzoq.z3flh4lzme66k@sl.local'], header_from:MAILER-DAEMON@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'MAILER-DAEMON@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'bounce'), ('Content-Type', 'multipart/report; report-type=delivery-status; boundary="b"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:26:54,019 - SL - DEBUG - 19695 - "/work/email_handler.py:1862" - handle_bounce() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - handle bounce for <EmailLog 360>, phase=forward, contact=<Contact 165 cc@example.com 775>, alias=<Alias 775 casing_forage632@sl.local>
2026-07-14 02:26:54,022 - SL - ERROR - 19695 - "/work/email_handler.py:1444" - handle_bounce_forward_phase() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - Use <Alias 775 casing_forage632@sl.local> default mailbox <Mailbox 537 user_kowbmfb0a5@mailbox.test>
NoneType: None
2026-07-14 02:26:54,022 - SL - WARNING - 19695 - "/work/email_handler.py:1453" - handle_bounce_forward_phase() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - cannot get bounce info, debug at
2026-07-14 02:26:54,024 - SL - DEBUG - 19695 - "/work/email_handler.py:1456" - handle_bounce_forward_phase() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - Handle forward bounce <Contact 165 cc@example.com 775> -> <Alias 775 casing_forage632@sl.local> -> <Mailbox 537 user_kowbmfb0a5@mailbox.test>. <EmailLog 360>
2026-07-14 02:26:54,029 - SL - WARNING - 19695 - "/work/email_handler.py:1474" - handle_bounce_forward_phase() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - Cannot parse original message from bounce message <Alias 775 casing_forage632@sl.local> <User 461 Test User user_kowbmfb0a5@mailbox.test> <Contact 165 cc@example.com 775> refused-emails/full-b2e42fe6-8e2d-48c5-b7e3-b1368bc1f85a.eml
2026-07-14 02:26:54,032 - SL - DEBUG - 19695 - "/work/email_handler.py:1491" - handle_bounce_forward_phase() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - Create refused email <Refused Email 120 None 2026-07-21T02:26:54.031645+00:00>
2026-07-14 02:26:54,044 - SL - DEBUG - 19695 - "/work/email_handler.py:1542" - handle_bounce_forward_phase() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - Inform user <User 461 Test User user_kowbmfb0a5@mailbox.test> about a bounce from contact <Contact 165 cc@example.com 775> to alias <Alias 775 casing_forage632@sl.local>
2026-07-14 02:26:54,080 - SL - DEBUG - 19695 - "/work/app/email_utils.py:303" - send_email() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - send email to user_kowbmfb0a5@mailbox.test, subject 'An email sent to casing_forage632@sl.local cannot be delivered to your mailbox'
2026-07-14 02:26:54,084 - SL - DEBUG - 19695 - "/work/app/mail_sender.py:131" - send() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - send email with subject 'An email sent to casing_forage632@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'user_kowbmfb0a5@mailbox.test'
2026-07-14 02:26:54,085 - SL - INFO - 19695 - "/work/email_handler.py:2367" - _handle() - 71778c78-3317-426d-9a8d-e7edf9e0bdcf - Finish mail_from <>, rcpt_tos ['sl.lmycyibtgyycyibsgm4dgmzugzoq.z3flh4lzme66k@sl.local'], takes 0.07318234443664551 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
HANDLER: handle_DATA() returned '250 SL E211 Bounce Forward phase handled'
AFTER  : email_log(id=360).bounced=True refused_email_id=120 bounced_mailbox_id=537 | Bounce_rows=116 RefusedEmail_rows=117
DELTA  : Bounce_rows +1, RefusedEmail_rows +1

===== GROUP-B OUTCOME SUMMARY (handle_DATA) =====
FWD OLD    VALID   OOO                               => 250 SL E214 Unauthorized for using reverse alias
FWD OLD    INVALID OOO                               => 550 SL E512 No such email log
FWD SIGNED VALID   OOO                               => 250 SL E214 Unauthorized for using reverse alias
FWD SIGNED INVALID OOO                               => 550 SL E512 No such email log
REP OLD    VALID   OOO                               => 250 Message accepted for delivery
REP OLD    INVALID OOO                               => 550 SL E512 No such email log
REP SIGNED VALID   OOO                               => 250 Message accepted for delivery
REP SIGNED INVALID OOO                               => 550 SL E512 No such email log
iCloud OLD VALID   mail_from                         => 250 SL E211 Bounce Forward phase handled
iCloud OLD INVALID mail_from                         => 550 SL E512 No such email log
STATE handle_DATA bounce                             => 250 SL E211 Bounce Forward phase handled
STATE bounced->True refused_id_set=True mbox_id_set=True Bounce+1 Refused+1 => committed
===== END SUMMARY (12 rows) =====
OBS_GROUP_B_DONE
```

**Executable two-run comparison of the OUTCOME SUMMARY blocks (status strings):**

```text
$ for r in gb_run1 gb_run2; do
    sed -n '/GROUP-B OUTCOME SUMMARY/,/END SUMMARY/p' $r.out > $r.sum
    echo "$r sha256: $(sha256sum $r.sum | cut -d' ' -f1)"
  done
gb_run1 sha256: fc1c4be7e906c5b0367a1914aaca7f65fade4b73ac7a96a6b186b8e4cdfba2bd
gb_run2 sha256: fc1c4be7e906c5b0367a1914aaca7f65fade4b73ac7a96a6b186b8e4cdfba2bd
$ diff -q gb_run1.sum gb_run2.sum && echo IDENTICAL
IDENTICAL
```

=> The two runs' OUTCOME SUMMARY blocks are **byte-identical** (both `sha256=fc1c4be7e906c5b0367a1914aaca7f65fade4b73ac7a96a6b186b8e4cdfba2bd`), so every Group-B status string is stable across at least two runs.

**State-transition lines (Part C) from both runs** — the concrete sequence-backed ids advance run-to-run (an `EmailLog`/`RefusedEmail`/`Bounce` sequence property of the accumulated test DB), but the observed **state semantics and deltas are identical**: `bounced` `False`->`True`, `refused_email_id`/`bounced_mailbox_id` `None`->set, and `+1`/`+1` row growth, with the handler returning `250 SL E211 Bounce Forward phase handled` both times:

```text
--- run 1 ---
BEFORE : email_log(id=354).bounced=False refused_email_id=None bounced_mailbox_id=None | Bounce_rows=113 RefusedEmail_rows=114
HANDLER: handle_DATA() returned '250 SL E211 Bounce Forward phase handled'
AFTER  : email_log(id=354).bounced=True refused_email_id=118 bounced_mailbox_id=534 | Bounce_rows=114 RefusedEmail_rows=115
DELTA  : Bounce_rows +1, RefusedEmail_rows +1
--- run 2 ---
BEFORE : email_log(id=360).bounced=False refused_email_id=None bounced_mailbox_id=None | Bounce_rows=115 RefusedEmail_rows=116
HANDLER: handle_DATA() returned '250 SL E211 Bounce Forward phase handled'
AFTER  : email_log(id=360).bounced=True refused_email_id=120 bounced_mailbox_id=537 | Bounce_rows=116 RefusedEmail_rows=117
DELTA  : Bounce_rows +1, RefusedEmail_rows +1
```

**What this addendum establishes, by objective:**

- **Objective 2 (routing) — forward OUT-OF-OFFICE:** a forward bounce address carrying an `Auto-Submitted: auto-replied` message (non-`<>` sender, `text/plain`, so `is_bounce` is `False` and `is_automatic_out_of_office` is `True`) falls through past `handle_out_of_office_forward_phase()` [email_handler.py:L2071-L2072] and the **observed** canonical response is `250 SL E214 Unauthorized for using reverse alias` for a valid id (OLD and signed alike), while an invalid id still returns `550 SL E512 No such email log`. E214 is the **observed** fall-through outcome — not inferred.
- **Objective 2 (routing) — reply OUT-OF-OFFICE:** the parallel reply branch falls through past `handle_out_of_office_reply_phase()` [email_handler.py:L2092-L2093]; the **observed** canonical response is `250 Message accepted for delivery` for a valid id (OLD and signed alike) and `550 SL E512 No such email log` for an invalid id.
- **iCloud branch (C.5) via `handle_DATA`:** a valid id in the envelope `mail_from` (`bounce+{id}+@sl.local`) with an ordinary non-bounce message returns `250 SL E211 Bounce Forward phase handled`; an invalid id returns `550 SL E512 No such email log` — the same `E211`/`E512` divergence observed via the direct-`handle()` probe in K.6, now confirmed through the canonical entry point.
- **State transition (D.4) via `handle_DATA`:** driving a genuine forward-phase bounce (`mail_from='<>'`, signed `bounce_forward`, `multipart/report`) through `handle_DATA` flips `EmailLog.bounced` `False`->`True`, sets `refused_email_id` and `bounced_mailbox_id`, and grows the `Bounce` and `RefusedEmail` tables by `+1` each — the committed mutation confirmed at the canonical entry point.



### K.11 Script VS — verifier-shape coverage (six return-None conditions of get_verp_info_from_email)

This appendix grounds Phase F.1's enumeration of the **six** `return None` conditions of `get_verp_info_from_email()` [app/email_utils.py:L1467-L1498] at runtime. It drives the real entry point `MailHandler.handle_DATA()` with malformed **signed-looking** recipients that trigger conditions **1-4** (no `@`; wrong prefix / short field list; non-base32 payload; corrupted HMAC) and reports both the verifier verdict (via a direct helper call, labelled **non-canonical**, for illustration) and the **canonical** `handle_DATA()` SMTP status. **No mock or patch is used anywhere in this script** — the malformed inputs are literal strings. Conditions **5** (`len(data) != 3`) and **6** (expiry) plus the `json.loads`-raise path require a payload that already passed the HMAC (i.e. produced by our own signer): condition 6 (`+6d` expiry) and condition 4 (corrupted signature) are additionally shown canonically in Phase K.4. The script was run twice; the OUTCOME SUMMARY is byte-identical across runs.

**Command:**

```text
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/obs_verifier_shapes.py
```

**Observation-script source (`obs_verifier_shapes.py`):**

```python
"""OBSERVATION SCRIPT (temporary, non-repository) — Objective 4 / D6 verifier-shape coverage.
Drives the REAL inbound path MailHandler.handle_DATA() with malformed signed-looking recipients that
trigger the verifier get_verp_info_from_email() return-None conditions 1-4 (source app/email_utils.py
L1467-L1498). For each, prints BOTH the verifier verdict via a DIRECT helper call (NON-CANONICAL, for
illustration) AND the CANONICAL handle_DATA() SMTP status. No mock/patch is used anywhere in this
script — the malformed inputs are literal strings. Conditions 5 (len(data)!=3) and 6 (expiry) plus the
json.loads-raise path require a payload that already passed the HMAC (our own signer); condition 6
(+6d expiry) and condition 4 (corrupted HMAC) are additionally shown canonically in Phase K.4."""
import os, asyncio, sys
os.environ["CONFIG"] = "/work/tests/test.env"
from aiosmtpd.smtp import Envelope
import aiosmtpd
from app.db import Session, engine, connection
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app(); app.config["TESTING"] = True; app.config["SERVER_NAME"] = "sl.test"
with engine.connect() as conn:
    try: conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    except Exception: conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
import email_handler
from app import config
from app.models import Alias, EmailLog, Contact, VerpType
from app.email_utils import generate_verp_email, get_verp_info_from_email
from tests.utils import create_new_user

NONBOUNCE = ("From: attacker@evil.example\r\nTo: probe@sl.local\r\nSubject: hi\r\n"
             "Content-Type: text/plain\r\n\r\nhello\r\n")

def run_handle_data(mail_from, rcpt, raw):
    env = Envelope(); env.mail_from = mail_from; env.rcpt_tos = [rcpt]
    env.original_content = raw.encode()
    return asyncio.run(email_handler.MailHandler().handle_DATA(None, None, env))

def verdict(addr):
    try: return repr(get_verp_info_from_email(addr))
    except Exception as e: return "RAISED %s(%s)" % (type(e).__name__, e)

pgver = Session.execute("SELECT version()").scalar().split(",")[0]
print("=== ENV: %s | python %s | aiosmtpd %s ===\n" % (pgver, sys.version.split()[0], aiosmtpd.__version__))

summary = []
transaction = connection.begin()
with app.app_context():
    config.DISABLE_RATE_LIMIT = True
    try:
        # a genuine signed address to derive a corrupted-HMAC variant (condition 4)
        u = create_new_user(); a = Alias.create_new_random(u); Session.commit()
        c = Contact.create(user_id=u.id, alias_id=a.id, website_email="c@example.com", reply_email="rep@sl.local", commit=True)
        el = EmailLog.create(user_id=u.id, contact_id=c.id, alias_id=a.id, is_reply=False, commit=True)
        good = generate_verp_email(VerpType.bounce_forward, el.id)
        lp, dom = good.split("@"); f = lp.split(".")
        sig = list(f[2]); sig[0] = ("b" if sig[0] != "b" else "c")   # flip 1 char in base32 signature
        corrupted = "%s.%s.%s@%s" % (f[0], f[1], "".join(sig), dom)

        cases = [
          ("cond1 no-@                 ", "sl-no-at-sign-here"),
          ("cond2 wrong-prefix         ", "notsl.aaaaaaaa.bbbbbbbb@sl.local"),
          ("cond2 short-field-list     ", "sl.onlyonefield@sl.local"),
          ("cond3 non-base32 payload   ", "sl.11111111.99999999@sl.local"),
          ("cond4 corrupted-HMAC       ", corrupted),
        ]
        for label, rcpt in cases:
            v = verdict(rcpt)
            r = run_handle_data("attacker@evil.example", rcpt, NONBOUNCE)
            print("%s rcpt=%r\n   get_verp_info_from_email() [NON-CANONICAL direct] = %s\n   handle_DATA() [CANONICAL] = %r"
                  % (label, rcpt, v, r))
            summary.append((label.strip(), v, r))
    finally:
        transaction.rollback(); Session.rollback(); Session.close()

print("\n===== VERIFIER-SHAPE OUTCOME SUMMARY =====")
for lbl, v, r in summary:
    print("%-26s verifier=%-6s handle_DATA=%s" % (lbl, v, r))
print("===== END SUMMARY (%d rows) =====" % len(summary))
print("OBS_VERIFIER_SHAPES_DONE")
```

**Complete verbatim transcript — RUN 1 (`vs_run1.out`):**

```text
load config file /work/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/cznxlcmiuklualmzgjqg
Upload files to local dir
>>> init logging <<<
2026-07-14 02:44:30,814 - SL - DEBUG - 20714 - "/work/app/utils.py:17" - <module>() -  - load words file: /work/local_data/test_words.txt
2026-07-14 02:44:31,990 - SL - DEBUG - 20714 - "/work/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-14 02:44:31,991 - SL - DEBUG - 20714 - "/work/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-14 02:44:31,992 - SL - DEBUG - 20714 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 02:44:31,993 - SL - DEBUG - 20714 - "/work/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
=== ENV: PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu | python 3.10.18 | aiosmtpd 1.4.2 ===

2026-07-14 02:44:32,313 - SL - INFO - 20714 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:44:32,329 - SL - DEBUG - 20714 - "/work/app/models.py:1459" - generate_random_alias_email() -  - generate email bovver_dweebs865@sl.local
2026-07-14 02:44:32,339 - SL - INFO - 20714 - "/work/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 02:44:32,363 - SL - DEBUG - 20714 - "/work/app/log.py:24" - set_message_id() -  - set message_id 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82
2026-07-14 02:44:32,363 - SL - DEBUG - 20714 - "/work/email_handler.py:2342" - _handle() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:44:32,363 - SL - INFO - 20714 - "/work/email_handler.py:2343" - _handle() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - New message, mail from attacker@evil.example, rctp tos ['sl-no-at-sign-here']
2026-07-14 02:44:32,364 - SL - INFO - 20714 - "/work/email_handler.py:1956" - handle() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:44:32,364 - SL - DEBUG - 20714 - "/work/email_handler.py:1963" - handle() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - Cannot parse Postfix queue ID from None None
2026-07-14 02:44:32,365 - SL - DEBUG - 20714 - "/work/email_handler.py:1980" - handle() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl-no-at-sign-here'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:44:32,370 - SL - DEBUG - 20714 - "/work/email_handler.py:2202" - handle() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - Forward phase attacker@evil.example(attacker@evil.example) -> sl-no-at-sign-here
2026-07-14 02:44:32,378 - SL - DEBUG - 20714 - "/work/email_handler.py:545" - handle_forward() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - alias sl-no-at-sign-here not exist. Try to see if it can be created on the fly
2026-07-14 02:44:32,378 - SL - DEBUG - 20714 - "/work/email_handler.py:551" - handle_forward() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - alias sl-no-at-sign-here cannot be created on-the-fly, return 550
2026-07-14 02:44:32,379 - SL - INFO - 20714 - "/work/email_handler.py:2367" - _handle() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - Finish mail_from attacker@evil.example, rcpt_tos ['sl-no-at-sign-here'], takes 0.016396284103393555 seconds with return code '550 SL E515 Email not exist'<<===
cond1 no-@                  rcpt='sl-no-at-sign-here'
   get_verp_info_from_email() [NON-CANONICAL direct] = None
   handle_DATA() [CANONICAL] = '550 SL E515 Email not exist'
2026-07-14 02:44:32,380 - SL - DEBUG - 20714 - "/work/app/log.py:24" - set_message_id() - 9f1c6c29-bd8a-49b1-b664-b0d06e5c2d82 - set message_id 016cbd35-d875-4760-9799-c00dace8b655
2026-07-14 02:44:32,380 - SL - DEBUG - 20714 - "/work/email_handler.py:2342" - _handle() - 016cbd35-d875-4760-9799-c00dace8b655 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:44:32,380 - SL - INFO - 20714 - "/work/email_handler.py:2343" - _handle() - 016cbd35-d875-4760-9799-c00dace8b655 - New message, mail from attacker@evil.example, rctp tos ['notsl.aaaaaaaa.bbbbbbbb@sl.local']
2026-07-14 02:44:32,381 - SL - INFO - 20714 - "/work/email_handler.py:1956" - handle() - 016cbd35-d875-4760-9799-c00dace8b655 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:44:32,381 - SL - DEBUG - 20714 - "/work/email_handler.py:1963" - handle() - 016cbd35-d875-4760-9799-c00dace8b655 - Cannot parse Postfix queue ID from None None
2026-07-14 02:44:32,382 - SL - DEBUG - 20714 - "/work/email_handler.py:1980" - handle() - 016cbd35-d875-4760-9799-c00dace8b655 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['notsl.aaaaaaaa.bbbbbbbb@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:44:32,386 - SL - DEBUG - 20714 - "/work/email_handler.py:2202" - handle() - 016cbd35-d875-4760-9799-c00dace8b655 - Forward phase attacker@evil.example(attacker@evil.example) -> notsl.aaaaaaaa.bbbbbbbb@sl.local
2026-07-14 02:44:32,394 - SL - DEBUG - 20714 - "/work/email_handler.py:545" - handle_forward() - 016cbd35-d875-4760-9799-c00dace8b655 - alias notsl.aaaaaaaa.bbbbbbbb@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 02:44:32,400 - SL - INFO - 20714 - "/work/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 016cbd35-d875-4760-9799-c00dace8b655 - Cannot auto-create custom domain alias for notsl.aaaaaaaa.bbbbbbbb@sl.local because there's no custom domain for sl.local
2026-07-14 02:44:32,400 - SL - INFO - 20714 - "/work/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 016cbd35-d875-4760-9799-c00dace8b655 - Cannot auto-create notsl.aaaaaaaa.bbbbbbbb@sl.local since it has no directory separator
2026-07-14 02:44:32,400 - SL - DEBUG - 20714 - "/work/email_handler.py:551" - handle_forward() - 016cbd35-d875-4760-9799-c00dace8b655 - alias notsl.aaaaaaaa.bbbbbbbb@sl.local cannot be created on-the-fly, return 550
2026-07-14 02:44:32,401 - SL - INFO - 20714 - "/work/email_handler.py:2367" - _handle() - 016cbd35-d875-4760-9799-c00dace8b655 - Finish mail_from attacker@evil.example, rcpt_tos ['notsl.aaaaaaaa.bbbbbbbb@sl.local'], takes 0.02069997787475586 seconds with return code '550 SL E515 Email not exist'<<===
cond2 wrong-prefix          rcpt='notsl.aaaaaaaa.bbbbbbbb@sl.local'
   get_verp_info_from_email() [NON-CANONICAL direct] = None
   handle_DATA() [CANONICAL] = '550 SL E515 Email not exist'
2026-07-14 02:44:32,402 - SL - DEBUG - 20714 - "/work/app/log.py:24" - set_message_id() - 016cbd35-d875-4760-9799-c00dace8b655 - set message_id 4a66ccde-8875-473c-9e16-3aa6fdea1e6b
2026-07-14 02:44:32,402 - SL - DEBUG - 20714 - "/work/email_handler.py:2342" - _handle() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - ====>=====>====>====>====>====>====>====>
2026-07-14 02:44:32,402 - SL - INFO - 20714 - "/work/email_handler.py:2343" - _handle() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - New message, mail from attacker@evil.example, rctp tos ['sl.onlyonefield@sl.local']
2026-07-14 02:44:32,403 - SL - INFO - 20714 - "/work/email_handler.py:1956" - handle() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:44:32,403 - SL - DEBUG - 20714 - "/work/email_handler.py:1963" - handle() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - Cannot parse Postfix queue ID from None None
2026-07-14 02:44:32,404 - SL - DEBUG - 20714 - "/work/email_handler.py:1980" - handle() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.onlyonefield@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:44:32,407 - SL - DEBUG - 20714 - "/work/email_handler.py:2202" - handle() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - Forward phase attacker@evil.example(attacker@evil.example) -> sl.onlyonefield@sl.local
2026-07-14 02:44:32,414 - SL - DEBUG - 20714 - "/work/email_handler.py:545" - handle_forward() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - alias sl.onlyonefield@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 02:44:32,419 - SL - INFO - 20714 - "/work/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - Cannot auto-create custom domain alias for sl.onlyonefield@sl.local because there's no custom domain for sl.local
2026-07-14 02:44:32,419 - SL - INFO - 20714 - "/work/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - Cannot auto-create sl.onlyonefield@sl.local since it has no directory separator
2026-07-14 02:44:32,419 - SL - DEBUG - 20714 - "/work/email_handler.py:551" - handle_forward() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - alias sl.onlyonefield@sl.local cannot be created on-the-fly, return 550
2026-07-14 02:44:32,419 - SL - INFO - 20714 - "/work/email_handler.py:2367" - _handle() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - Finish mail_from attacker@evil.example, rcpt_tos ['sl.onlyonefield@sl.local'], takes 0.017359256744384766 seconds with return code '550 SL E515 Email not exist'<<===
cond2 short-field-list      rcpt='sl.onlyonefield@sl.local'
   get_verp_info_from_email() [NON-CANONICAL direct] = None
   handle_DATA() [CANONICAL] = '550 SL E515 Email not exist'
2026-07-14 02:44:32,420 - SL - DEBUG - 20714 - "/work/app/log.py:24" - set_message_id() - 4a66ccde-8875-473c-9e16-3aa6fdea1e6b - set message_id 9200e09b-7647-4eb3-9f1e-2a1b6f36208f
2026-07-14 02:44:32,420 - SL - DEBUG - 20714 - "/work/email_handler.py:2342" - _handle() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - ====>=====>====>====>====>====>====>====>
2026-07-14 02:44:32,420 - SL - INFO - 20714 - "/work/email_handler.py:2343" - _handle() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - New message, mail from attacker@evil.example, rctp tos ['sl.11111111.99999999@sl.local']
2026-07-14 02:44:32,421 - SL - INFO - 20714 - "/work/email_handler.py:1956" - handle() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:44:32,421 - SL - DEBUG - 20714 - "/work/email_handler.py:1963" - handle() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - Cannot parse Postfix queue ID from None None
2026-07-14 02:44:32,422 - SL - DEBUG - 20714 - "/work/email_handler.py:1980" - handle() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.11111111.99999999@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:44:32,425 - SL - DEBUG - 20714 - "/work/email_handler.py:2202" - handle() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - Forward phase attacker@evil.example(attacker@evil.example) -> sl.11111111.99999999@sl.local
2026-07-14 02:44:32,431 - SL - DEBUG - 20714 - "/work/email_handler.py:545" - handle_forward() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - alias sl.11111111.99999999@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 02:44:32,435 - SL - INFO - 20714 - "/work/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - Cannot auto-create custom domain alias for sl.11111111.99999999@sl.local because there's no custom domain for sl.local
2026-07-14 02:44:32,435 - SL - INFO - 20714 - "/work/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - Cannot auto-create sl.11111111.99999999@sl.local since it has no directory separator
2026-07-14 02:44:32,435 - SL - DEBUG - 20714 - "/work/email_handler.py:551" - handle_forward() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - alias sl.11111111.99999999@sl.local cannot be created on-the-fly, return 550
2026-07-14 02:44:32,435 - SL - INFO - 20714 - "/work/email_handler.py:2367" - _handle() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - Finish mail_from attacker@evil.example, rcpt_tos ['sl.11111111.99999999@sl.local'], takes 0.015340328216552734 seconds with return code '550 SL E515 Email not exist'<<===
cond3 non-base32 payload    rcpt='sl.11111111.99999999@sl.local'
   get_verp_info_from_email() [NON-CANONICAL direct] = None
   handle_DATA() [CANONICAL] = '550 SL E515 Email not exist'
2026-07-14 02:44:32,436 - SL - DEBUG - 20714 - "/work/app/log.py:24" - set_message_id() - 9200e09b-7647-4eb3-9f1e-2a1b6f36208f - set message_id 5da310e0-8ea9-49bb-8653-541796ee4a22
2026-07-14 02:44:32,436 - SL - DEBUG - 20714 - "/work/email_handler.py:2342" - _handle() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - ====>=====>====>====>====>====>====>====>
2026-07-14 02:44:32,436 - SL - INFO - 20714 - "/work/email_handler.py:2343" - _handle() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - New message, mail from attacker@evil.example, rctp tos ['sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local']
2026-07-14 02:44:32,437 - SL - INFO - 20714 - "/work/email_handler.py:1956" - handle() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 02:44:32,437 - SL - DEBUG - 20714 - "/work/email_handler.py:1963" - handle() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - Cannot parse Postfix queue ID from None None
2026-07-14 02:44:32,438 - SL - DEBUG - 20714 - "/work/email_handler.py:1980" - handle() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - ==>> Handle mail_from:attacker@evil.example, rcpt_tos:['sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local'], header_from:attacker@evil.example, header_to:probe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'attacker@evil.example'), ('To', 'probe@sl.local'), ('Subject', 'hi'), ('Content-Type', 'text/plain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 02:44:32,441 - SL - DEBUG - 20714 - "/work/email_handler.py:2202" - handle() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - Forward phase attacker@evil.example(attacker@evil.example) -> sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local
2026-07-14 02:44:32,447 - SL - DEBUG - 20714 - "/work/email_handler.py:545" - handle_forward() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - alias sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 02:44:32,450 - SL - INFO - 20714 - "/work/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - Cannot auto-create custom domain alias for sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local because there's no custom domain for sl.local
2026-07-14 02:44:32,450 - SL - INFO - 20714 - "/work/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - Cannot auto-create sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local since it has no directory separator
2026-07-14 02:44:32,450 - SL - DEBUG - 20714 - "/work/email_handler.py:551" - handle_forward() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - alias sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local cannot be created on-the-fly, return 550
2026-07-14 02:44:32,451 - SL - INFO - 20714 - "/work/email_handler.py:2367" - _handle() - 5da310e0-8ea9-49bb-8653-541796ee4a22 - Finish mail_from attacker@evil.example, rcpt_tos ['sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local'], takes 0.014757156372070312 seconds with return code '550 SL E515 Email not exist'<<===
cond4 corrupted-HMAC        rcpt='sl.lmycyibtgy3syibsgm4dgmzwgroq.bza2qzfzefyj2@sl.local'
   get_verp_info_from_email() [NON-CANONICAL direct] = None
   handle_DATA() [CANONICAL] = '550 SL E515 Email not exist'

===== VERIFIER-SHAPE OUTCOME SUMMARY =====
cond1 no-@                 verifier=None   handle_DATA=550 SL E515 Email not exist
cond2 wrong-prefix         verifier=None   handle_DATA=550 SL E515 Email not exist
cond2 short-field-list     verifier=None   handle_DATA=550 SL E515 Email not exist
cond3 non-base32 payload   verifier=None   handle_DATA=550 SL E515 Email not exist
cond4 corrupted-HMAC       verifier=None   handle_DATA=550 SL E515 Email not exist
===== END SUMMARY (5 rows) =====
OBS_VERIFIER_SHAPES_DONE
```

**Two-run stability (OUTCOME SUMMARY SHA-256):**

```text
vs_run1 sha256: 5a739e69fd656b5fcf23c89aa87af14d432b70ea5a695081122a085d089c3a4a
vs_run2 sha256: 5a739e69fd656b5fcf23c89aa87af14d432b70ea5a695081122a085d089c3a4a
$ diff -q vs_run1.sum vs_run2.sum && echo IDENTICAL
IDENTICAL
```

=> All four attacker-reachable verifier conditions (1-4) return `None` from `get_verp_info_from_email()`, and the canonical `handle_DATA()` response for each malformed signed-looking recipient is `550 SL E515 Email not exist` — the signed path rejects the forgery permanently, in contrast to the unsigned `bounce+{id}+@` path which performs no such check. This is stable across two runs (`sha256=5a739e69fd656b5fcf23c89aa87af14d432b70ea5a695081122a085d089c3a4a`).
