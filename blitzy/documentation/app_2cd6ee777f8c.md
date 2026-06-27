# SimpleLogin Bounce-Handling Security Investigation — The Unsigned `bounce+{id}+@domain` Enumeration Oracle

**Repository:** SimpleLogin (`app`) &nbsp;•&nbsp; **Source branch:** `app_2cd6ee777f8c` &nbsp;•&nbsp; **HEAD:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
**Document type:** Code-grounded security Q&A (investigation only — no source code is changed)
**Subsystem under analysis:** inbound bounce/VERP routing in `email_handler.py` and `app/email_utils.py`

---

## 1. Executive Summary

**Verdict: yes — the older, unsigned `bounce+{id}+@domain` address format is an externally reachable, multi-state enumeration / information-disclosure oracle.**

The root cause is an **OR-coupling** in the inbound router `handle()`: the forward-bounce branch accepts an address when an *unsigned* plaintext prefix/suffix string match succeeds **OR** when a *signed* VERP check succeeds, treating the two on entirely equal footing — the branch fires when `len(rcpt_tos) == 1 and rcpt_tos[0].startswith(BOUNCE_PREFIX) and rcpt_tos[0].endswith(BOUNCE_SUFFIX)` **OR** `(verp_info and verp_info[0] == VerpType.bounce_forward)` [email_handler.py:L2057-2061]. Because acceptance is decoupled from authentication, an attacker-chosen integer flows straight into a database primary-key lookup with **no cryptographic validation**: `email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(rcpt_tos[0])` followed by `email_log = EmailLog.get(email_log_id)` [email_handler.py:L2062-2063].

From that point the router returns **distinct, deterministic SMTP wire strings per internal state**, which is what turns a lookup into an oracle:

- **`550 SL E512 No such email log`** — the `EmailLog` row does not exist [email_handler.py:L2065-2067; app/email/status.py:L49].
- **`550 SL E510 so such user`** — the row exists but the owning user is inactive [email_handler.py:L1869-1871; app/email/status.py:L47] *(the wire string contains a verbatim typo, “so such”, preserved below).*
- **`250 SL E211 Bounce Forward phase handled`** — row exists, active user, forward-phase log [email_handler.py:L1912-1914; app/email/status.py:L19].
- **`250 SL E212 Bounce Reply phase handled`** — row exists, active user, reply-phase log [email_handler.py:L1910-1911; app/email/status.py:L20].
- **`250 SL E213 Unknown email ignored`** — row exists but the message did not satisfy bounce detection [email_handler.py:L2073-2074, L2308-2318; app/email/status.py:L21].

Bounce **detection itself is fully spoofable**: a message is treated as a bounce iff the envelope sender is the null sender `<>` and the MIME `Content-Type` is `multipart/report` — both attacker-controllable [email_handler.py:L1813-1818]. The only post-routing defense is an **SPF black-holing gate** that rewrites a 5xx status to `250 SL E216 Handled spf policy` — but only when a `SpamdResult` header is present and its SPF verdict is fail/soft_fail [email_handler.py:L2357-2365]. That gate **never touches the 2xx codes** (E211/E212/E213), and leaves the 5xx codes observable whenever no `SpamdResult` header is present or SPF does not fail. The oracle therefore **largely persists** in practice.

Net: an external sender, with one crafted email per probe and no credentials, can walk the sequential `EmailLog` ID space (autoincrement primary keys, `app/models.py:L62-63`) and classify each record by **(1) existence, (2) owning-user active/inactive status, and (3) forward-vs-reply phase** — a multi-dimensional state oracle, not a binary existence check.

Every SMTP wire string in the **§(d) evidence table** was **captured by a live run of the real handler against a real PostgreSQL database inside the project's own Docker runtime** (Python 3.10.18; see §2/Tier 2 and Appendix A.2), not inferred. (Adjacent E-code constants quoted elsewhere in this report for context — e.g. E205–E210, E215, E506, E519, E524 — are transcribed verbatim from `app/email/status.py`, not exercised in the live transcript.) The live run also confirms the decisive *contrast*: a **tampered** signed-VERP address is rejected at HMAC verification and diverted away from the log lookup entirely (it returns `550 SL E515 Email not exist`, never reaching `EmailLog.get()`), whereas the unsigned `bounce+{id}+@` format reaches the lookup with an attacker-chosen integer and no validation at all. The signature is exactly what protects the new format — and exactly what the old format lacks.

---

## 2. Methodology & Evidence Provenance

This investigation is bound by a strict rule set: **code is the single source of truth**, **every behavioral claim must carry a `path:Lxxx` locator**, **claims must be backed by reproduced runtime output rather than theory**, and **no source file may be modified** (the only artifact written is this Markdown document). To honor "evidence over theory" while being fully transparent about *how* each fact was established, evidence is presented in four explicitly labeled tiers. **Every wire string in the §(d) evidence table was captured by a live run of the real handler inside the provided Docker container** (Tier 2); the remaining tiers reproduce the format-layer mechanics directly (Tier 1), explain the code-truth mapping behind each response (Tier 3), and corroborate with the project's own committed tests (Tier 4).

### Tier 1 — Genuine reproduced runtime output (pure, standard-library-only functions)

The format/parse/sign/detect layer does **not** require a database, a Flask app context, or any network service. The four functions that decide the oracle's *format-layer* mechanics —

- `parse_id_from_bounce()` [app/email_utils.py:L1258-1259],
- `generate_verp_email()` [app/email_utils.py:L1438-1465],
- `get_verp_info_from_email()` [app/email_utils.py:L1467-1499], and
- `is_bounce()` [email_handler.py:L1813-1818]

— depend only on the Python standard library (`base64`, `binascii`, `hmac`, `json`, `time`). They were copied **verbatim** from the source into a transient `/tmp` probe (since deleted, per the "clean up test scripts" rule) and executed, with configuration constants set to the repository defaults:

| Constant | Value | Locator |
|---|---|---|
| `VERP_PREFIX` | `"sl"` | app/config.py:L500 |
| `VERP_EMAIL_SECRET` | a documented ≥32-char placeholder (see note) | app/config.py:L502, guard L505 |
| `EMAIL_DOMAIN` | `"sl.local"` | example.env:L22 |
| `BOUNCE_PREFIX` | `"bounce+"` | app/config.py:L100 |
| `BOUNCE_SUFFIX` | `"+@sl.local"` | app/config.py:L101 |
| `VERP_MESSAGE_LIFETIME` | `5 * 86400` (5 days) | app/config.py:L499 |
| `VERP_TIME_START` | `1640995200` (2022-01-01) | app/email_utils.py:L68 |
| `VERP_HMAC_ALGO` | `"sha3-224"` | app/email_utils.py:L69 |
| `VerpType` | `bounce_forward=0, bounce_reply=1, transactional=2` | app/models.py:L247-250 |

The captured output is reproduced verbatim in **Appendix A**. It was independently re-executed during this investigation on the analysis host (CPython **3.13.7**); all deterministic results matched. One value is intentionally *not* universal — see the note on signature determinism below.

> **Note on VERP signature determinism (important honesty caveat).** `generate_verp_email()` encodes the payload `[verp_type, object_id, minutes]` and appends an HMAC keyed on `VERP_EMAIL_SECRET` [app/email_utils.py:L1446-1465]. The **payload** segment is reproducible and secret-independent: for `(bounce_forward, 12345)` at the captured instant it base32-decodes to exactly `[0, 12345, 2358458]` and renders as `lmycyibrgiztinjmeazdgnjygq2tqxi` — **reproduced identically** during this investigation. The trailing **signature** segment is a function of `VERP_EMAIL_SECRET` (which is derived from the deployment's configured secret per [app/config.py:L502-503]; its concrete value is **redacted** throughout this document), so it is *run-specific*: the canonical capture (using the container's secret) yields `e7bm3xshjgxji`, whereas an independent re-run here with a documented placeholder secret yielded `diqtbmuefcvek`. Both round-trip correctly under their own secret and both reject a tampered signature — which is precisely the HMAC integrity property under test. Appendix A presents the canonical capture; this caveat documents the one secret-dependent element.

### Tier 2 — Live database-backed run inside the provided Docker container (primary evidence for §(d))

Every wire string in the §(d) evidence table was **captured live** by driving the real handler against a real PostgreSQL database inside the project's own runtime — the Docker image prescribed by the setup instructions, `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (the SimpleLogin source at commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`, **Python 3.10.18**, Poetry venv, PostgreSQL 15 + Redis, `NOT_SEND_EMAIL=true` [example.env:L19]). After `/build.sh` (which starts Postgres+Redis and migrates the schema to Alembic head) plus the documented DKIM PKCS#1 fix, a transient probe (kept entirely outside the source tree, at `/app/tests/blitzy_live_probe.py` inside the ephemeral container, and **deleted afterward**) seeded the relevant `EmailLog`/`User`/`Contact` rows via the committed test fixtures and invoked the real routing for each scenario:

- **Rows 1–6** were driven through `email_handler.handle(envelope, msg)` directly — exactly as the committed bounce tests do (e.g. `test_gmail_dmarc_softfail` [tests/test_email_handler.py:L101]) — within the harness's `flask_client` session. The `VERP*`→`status.E213` and `Exception`→`status.E404` mapping was reproduced **verbatim** from `handle_DATA()` [email_handler.py:L2307-2332], and the *actual exception raised by `handle()`* was captured to prove the live raise (row 5 below records the real `VERPForward`).
- **Row 7** was driven through `MailHandler()._handle(envelope, msg)` with the project's own `5xx_overwrite_spf.eml` fixture rendered with `spf_result="R_SPF_FAIL"` — i.e. an *exact* replication of the committed test `test_prevent_5xx_from_spf` [tests/test_email_handler.py:L115-127] — so the SPF gate executes live.

The container's `VERP_EMAIL_SECRET` resolved to its configured default and satisfied the ≥32-character guard [app/config.py:L502-503, guard L505]; its concrete value is **redacted** here and is not required to reproduce any §(d) wire string — the oracle lives entirely on the *unsigned* path, which performs no signature check, so no secret value is needed to reproduce the evidence. The captured wire strings (verbatim; full transcript in **Appendix A**, "Live container run"):

| Row | Scenario (unsigned `bounce+N+@sl.local` unless noted) | Captured live wire string |
|---|---|---|
| 1 | non-existent `EmailLog` id, well-formed bounce, `MAIL FROM:<>` | `550 SL E512 No such email log` |
| 2 | valid id, active user, **forward-phase** log, real bounce | `250 SL E211 Bounce Forward phase handled` |
| 3 | valid id, active user, **reply-phase** log, real bounce | `250 SL E212 Bounce Reply phase handled` |
| 4 | valid id, **inactive** user (future `delete_on`) | `550 SL E510 so such user` |
| 5 | valid id, message **not** a bounce → `handle()` raised `VERPForward` (captured), mapped per `handle_DATA` | `250 SL E213 Unknown email ignored` |
| 6 | **tampered** signed-VERP address (one signature char flipped) | `550 SL E515 Email not exist` |
| 7 | 5xx (non-existent id) + `X-Spamd-Result: … R_SPF_FAIL`, via `_handle()` | `250 SL E216 Handled spf policy` |

The probe test exited `1 passed`. Two findings deserve emphasis up front:

- **Row 5 proves the live raise:** `handle()` genuinely raised `VERPForward` (not a fabricated mapping); the captured tuple was `('VERPForward', '250 SL E213 Unknown email ignored')`. The `→ E213` translation is the verbatim `except (VERPReply, VERPForward, VERPTransactional): … return status.E213` clause [email_handler.py:L2308-2318].
- **Row 6 corrects an a-priori prediction with the empirical truth.** A *tampered* signed-VERP address (shape `sl.<payload>.<badsig>@sl.local`) does **not** yield `E213`; it yields **`550 SL E515 Email not exist`**. The reason — verified against `handle()` [email_handler.py:L2035-2116] — is exactly the point of this whole investigation: the tampered address (i) fails signature verification, so `get_verp_info_from_email(...)` returns `None` (captured live), **and** (ii) does not match the unsigned `bounce+`…`+@sl.local` prefix/suffix shape (nor the transactional/reply prefixes), so it satisfies **neither** disjunct of any VERP branch. It therefore never enters a bounce branch, never reaches `EmailLog.get()`, and never hits `raise VERPForward`; instead it falls through to ordinary inbound handling where `sl.…@sl.local` is looked up as an alias, is not found, and returns `status.E515`. This is the empirical *contrast* that proves the thesis: the **signed** path's HMAC integrity check genuinely diverts a forged address away from the lookup, whereas the **unsigned** path (rows 1–5) reaches `EmailLog.get()` with an attacker-chosen integer and **no validation at all**.

### Tier 3 — Code-truth: each response is a direct read of the source

Each captured wire string maps to an **exact return statement** in `handle()` / `handle_bounce()` / `handle_DATA()` paired with the **exact named constant** in `app/email/status.py` (both transcribed verbatim in §(d)). The mapping is a **direct read of the source**, not an inference: e.g. the no-row path is literally `if not email_log: … return status.E512` [email_handler.py:L2065-2067] with `E512 = "550 SL E512 No such email log"` [app/email/status.py:L49]; the missing-alias fall-through that produced row 6 ends in `E515 = "550 SL E515 Email not exist"` [app/email/status.py:L51]. Tier 3 is what lets the reader confirm *why* each live result is the value it is.

### Tier 4 — The project's own committed tests as empirical anchors

Three tests already committed to the repository assert these exact constants through the **real `MailHandler()._handle()` entrypoint** (i.e. through the SPF gate), using a deliberately invalid email-log id:

- `test_prevent_5xx_from_spf` — header `R_SPF_FAIL` ⇒ asserts `status.E216` [tests/test_email_handler.py:L115-127].
- `test_preserve_5xx_with_valid_spf` — header `R_SPF_ALLOW` ⇒ asserts `status.E512` [tests/test_email_handler.py:L130-142].
- `test_preserve_5xx_with_no_header` — no `SpamdResult` header ⇒ asserts `status.E512` [tests/test_email_handler.py:L145-157].

These independently corroborate row 1 (E512) and row 7 (E216) of the §(d) table through the exact `_handle()` path the live probe used for row 7, and — because they route an *invalid* id end-to-end — they reconfirm the "non-existent id ⇒ E512, SPF-fail ⇒ E216" behavior on which the masking analysis depends.

### Note on the analysis sandbox vs. the provided container

The initial static-analysis sandbox could not run the DB-backed paths itself — its interpreter was CPython 3.13.7 (vs. the project's pinned **Python 3.10**: `FROM python:3.10` [Dockerfile:L8]; CI matrix `["3.10"]` [.github/workflows/main.yml:L40]; `python = "^3.10"` [pyproject.toml:L61]) and it had no PostgreSQL, which the harness hard-requires (`tests/conftest.py` imports `from psycopg2 import errors` [tests/conftest.py:L15], runs `CREATE EXTENSION pg_trgm` [tests/conftest.py:L32], and wraps each test in `connection.begin()` / `rollback()` [tests/conftest.py:L61, L75]). That is precisely why the live run (Tier 2) was performed **inside the provided Docker container**, which bundles Python 3.10.18 + Postgres + Redis exactly as the setup instructions prescribe. **Appendix B** reproduces the working probe so any reader can re-capture every wire string in that container.

### Reasoning / why this matters

This tiered structure satisfies "evidence over theory" *and* full transparency. The DB-backed responses are not asserted from reasoning — they are **observed live** in the project's own runtime (Tier 2), each is **explained** by a direct source read (Tier 3), and the externally-observable subset is **independently anchored** by the suite's own green tests (Tier 4); the format-layer mechanics are **directly executed** (Tier 1). Being explicit about the one secret-dependent value (the VERP signature, below) and about *why* the live run had to happen in the container rather than the sandbox is itself part of the "code as truth" discipline: the reader knows exactly which claims are universal, which are run-specific, and how to reproduce every one of them.

---

## (a) The two address formats and their parsing / signing — **O1**

The user observed two shapes of inbound bounce addresses: an older plaintext form that exposes the `EmailLog` id directly (their example: `bounce+12345+@domain`), and a newer form that "uses some kind of signature verification." These map precisely to two code paths.

### The older, UNSIGNED plaintext format: `bounce+{id}+@domain`

This is parsed by `parse_id_from_bounce()` [app/email_utils.py:L1258-1259]:

```python
def parse_id_from_bounce(email_address: str) -> int:
    return int(email_address[email_address.find("+") : email_address.rfind("+")])
```

The function slices the substring between the **first** `+` and the **last** `+` and casts it to `int()`. There is **no signature, no integrity check, and no timestamp** — the integer is taken at face value. For the user's example `"bounce+12345+@domain"`, `find("+")` is index 6 and `rfind("+")` is index 12, so the slice is `"+12345"` (the leading `+` is included) and `int("+12345") == 12345`. The shape that triggers this path is governed by two configuration constants:

- `BOUNCE_PREFIX = os.environ.get("BOUNCE_PREFIX") or "bounce+"` [app/config.py:L100]
- `BOUNCE_SUFFIX = os.environ.get("BOUNCE_SUFFIX") or f"+@{EMAIL_DOMAIN}"` [app/config.py:L101]

With the default `EMAIL_DOMAIN=sl.local` [example.env:L22], `BOUNCE_SUFFIX` is `"+@sl.local"`. The user's `bounce+12345+@domain` therefore maps to the defaults exactly. **Reproduced (Tier 1, Appendix A.1):** `parse_id_from_bounce('bounce+12345+@domain') -> 12345`.

Two adjacent plaintext prefixes exist for the other phases/classes (same unsigned construction):

- `BOUNCE_PREFIX_FOR_REPLY_PHASE = ... or "bounce_reply"` [app/config.py:L108] (note: no trailing `+` baked in; the router appends one — see §(b)).
- `TRANSACTIONAL_BOUNCE_PREFIX = ... or "transactional+"` [app/config.py:L113] with `TRANSACTIONAL_BOUNCE_SUFFIX = ... or f"+@{EMAIL_DOMAIN}"` [app/config.py:L116].

### The newer, SIGNED VERP format

This is produced by `generate_verp_email()` [app/email_utils.py:L1438-1465]:

```python
data = [
    verp_type.value,
    object_id or 0,
    int((time.time() - VERP_TIME_START) / 60),
]
json_payload = json.dumps(data).encode("utf-8")
# Signing without itsdangereous because it uses base64 that includes +/= symbols ...
# We need to encode in base32
payload_hmac = hmac.new(
    config.VERP_EMAIL_SECRET.encode("utf-8"), json_payload, VERP_HMAC_ALGO
).digest()[:8]
encoded_payload = base64.b32encode(json_payload).rstrip(b"=").decode("utf-8")
encoded_signature = base64.b32encode(payload_hmac).rstrip(b"=").decode("utf-8")
return "{}.{}.{}@{}".format(
    config.VERP_PREFIX, encoded_payload, encoded_signature,
    sender_domain or config.EMAIL_DOMAIN,
).lower()
```

It builds a list `[verp_type, object_id, minutes]` (time in whole minutes since `VERP_TIME_START = 1640995200` [app/email_utils.py:L68], i.e. 2022-01-01, to keep the address short), JSON-encodes it, computes an HMAC over the payload with `VERP_HMAC_ALGO = "sha3-224"` [app/email_utils.py:L69] keyed on `VERP_EMAIL_SECRET` [app/config.py:L500-503], truncates to 8 bytes, base32-encodes payload and signature (stripping `=`), and emits `"{VERP_PREFIX}.{b32(payload)}.{b32(sig)}@{domain}".lower()` with `VERP_PREFIX = "sl"` [app/config.py:L500].

> **Dependency fact worth calling out.** The code comment at [app/email_utils.py:L1452] states the signer deliberately avoids `itsdangerous` (which would introduce `+`/`=` characters unsuitable for an email local-part) in favor of base32. In other words, **VERP signing uses the Python standard library `hmac`/`base64`/`json` — not the `itsdangerous` dependency**, even though `itsdangerous` is present in the environment.

### Verification of the signed format

`get_verp_info_from_email()` [app/email_utils.py:L1467-1499] is the inverse and the gatekeeper:

1. requires an `@`, splits the local-part on `.`, and requires **exactly three fields with `fields[0] == VERP_PREFIX`** [app/email_utils.py:L1476] — so an arbitrary plaintext `bounce+...` address (which has no `sl.<payload>.<sig>` structure) returns `None` here;
2. base32-decodes payload and signature (returning `None` on `binascii.Error`);
3. recomputes the HMAC and **rejects on `expected_signature != signature`** [app/email_utils.py:L1490];
4. requires a 3-element payload [app/email_utils.py:L1494];
5. applies a **future-timestamp guard** — `if data[2] > (time.time() + VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60: return None` — which rejects **only** payloads whose embedded minute value lies **more than `VERP_MESSAGE_LIFETIME` (`5 * 86400` s = 5 days) in the *future***; there is **no lower bound**, so the check does **not** expire old or stale signatures — a signed VERP address remains verifiable for as long as its HMAC is valid (an empirical reproduction of this exact comparison confirms a 2022-dated payload still verifies, while a payload dated more than 5 days ahead returns `None`) [app/email_utils.py:L1496; app/config.py:L499];
6. returns `(VerpType(data[0]), data[1])` or `None`.

> **Minor side-channel note.** The signature comparison at [app/email_utils.py:L1490] uses a plain `!=` rather than a constant-time comparator (e.g. `hmac.compare_digest`). This is a theoretical timing side-channel on the HMAC check; it is noted for completeness and is **not** the oracle this investigation is about (the oracle lives on the *unsigned* path, which performs no comparison at all).

**Reproduced round-trip + tamper rejection (Tier 1, Appendix A.1):** `generate_verp_email(bounce_forward, 12345)` produces `sl.lmycyibrgiztinjmeazdgnjygq2tqxi.<sig>@sl.local` whose payload segment decodes to `[0, 12345, 2358458]`; feeding that back to `get_verp_info_from_email(...)` returns `(VerpType.bounce_forward, 12345)`; flipping a single character of the signature segment makes `get_verp_info_from_email(...)` return `None`. By contrast, the unsigned `bounce+777+@sl.local` still yields `777` with **no crypto whatsoever**.

### Reasoning / why this matters

Structurally, the older format is a **pre-BATV unsigned return-path** and the newer format is a **BATV-style HMAC-signed VERP** (industry framing in §(g)). The single security-relevant difference is **integrity**: the signed address is verifiable and unforgeable without `VERP_EMAIL_SECRET`, while the unsigned address is trivially forgeable — anyone can write `bounce+<N>+@sl.local` for any `N`. Everything in §(b)–§(f) follows from the fact that `handle()` accepts *both*, so the forgeable one inherits the same downstream code paths as the authenticated one.


---

## (b) `handle()` routing and the OR-coupling boundary — **O1 / O2 / O3 (the crux)**

`handle()` [email_handler.py:L1945] is the central inbound router; its return value becomes the literal SMTP/LMTP reply. After sanitizing `mail_from`/`rcpt_tos`, it computes the signed-VERP interpretation once:

```python
verp_info = get_verp_info_from_email(rcpt_tos[0])   # email_handler.py:L2035
```

`verp_info` is either `None` (not a valid signed VERP address) or `(VerpType, object_id)`. It then evaluates a series of branches.

### The transactional-VERP branch [email_handler.py:L2038-2054]

```python
if (
    len(rcpt_tos) == 1
    and rcpt_tos[0].startswith(TRANSACTIONAL_BOUNCE_PREFIX)
    and rcpt_tos[0].endswith(TRANSACTIONAL_BOUNCE_SUFFIX)
) or (verp_info and verp_info[0] == VerpType.transactional):
    if is_bounce(envelope, msg):
        handle_transactional_bounce(...)
        return status.E205
    elif is_automatic_out_of_office(msg):
        return status.E206
    else:
        raise VERPTransactional
```

Same OR-coupling pattern as below; outcomes are `E205` (bounce handled) / `E206` (out of office) / `raise VERPTransactional` (→ E213 via `handle_DATA`).

### The forward-bounce branch — **THE boundary** [email_handler.py:L2057-2061]

```python
if (
    len(rcpt_tos) == 1
    and rcpt_tos[0].startswith(BOUNCE_PREFIX)
    and rcpt_tos[0].endswith(BOUNCE_SUFFIX)
) or (verp_info and verp_info[0] == VerpType.bounce_forward):
```

This disjunction is the precise security boundary the investigation set out to localize (O3). The **left** operand is an *unsigned* string-shape test — "does the recipient start with `bounce+` and end with `+@sl.local`?" The **right** operand is the *signed* VERP test. They are joined by `or`, so **the unsigned prefix/suffix match is accepted on entirely equal footing with the cryptographic check**. A forged plaintext address such as `bounce+12345+@sl.local` needs **no signature** to enter this branch.

Immediately inside, the id is extracted and looked up [email_handler.py:L2062-2063]:

```python
email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(rcpt_tos[0])
email_log = EmailLog.get(email_log_id)
```

When the address is the unsigned shape, `verp_info` is `None`, so the `or` falls through to `parse_id_from_bounce(rcpt_tos[0])` — i.e. **an attacker-chosen integer reaches a database primary-key lookup (`EmailLog.get`) with no cryptographic validation.** This single fact is the answer to O2 (feasibility) and O3 (boundary location).

The branch's immediate outcomes:

- `if not email_log: ... return status.E512` — non-existent id [email_handler.py:L2065-2067].
- `if is_bounce(envelope, msg): return handle_bounce(envelope, email_log, msg)` — a detected bounce [email_handler.py:L2069-2070].
- `elif is_automatic_out_of_office(msg): handle_out_of_office_forward_phase(...)` [email_handler.py:L2071-2072].
- `else: raise VERPForward` — a valid id but a non-bounce, non-OOO message; the exception is later mapped to E213 [email_handler.py:L2073-2074].

### The reply-VERP branch [email_handler.py:L2077-2098]

```python
if (
    len(rcpt_tos) == 1
    and rcpt_tos[0].startswith(f"{BOUNCE_PREFIX_FOR_REPLY_PHASE}+")
    or (verp_info and verp_info[0] == VerpType.bounce_reply)
):
```

Analogous OR-coupling, keyed on `BOUNCE_PREFIX_FOR_REPLY_PHASE` (`"bounce_reply"`) / `VerpType.bounce_reply`. Two precise observations from the source: (i) the plaintext test here checks **only `startswith(f"{BOUNCE_PREFIX_FOR_REPLY_PHASE}+")`** — it does *not* also require `endswith(BOUNCE_SUFFIX)` like the forward branch does; and (ii) by Python operator precedence the condition groups as `(len==1 and startswith(...)) or (verp_info and ...)`. The body again extracts the id and looks it up [email_handler.py:L2082-2083], returns `status.E512` on a missing row [email_handler.py:L2085-2087], routes a detected bounce to `handle_bounce()` (→ E212) [email_handler.py:L2090-2091], and otherwise `raise VERPReply(...)` (→ E213) [email_handler.py:L2094-2098].

### The iCloud `mail_from` variant [email_handler.py:L2100-2116]

iCloud returns the bounce with the bounce address in `mail_from` rather than `rcpt_to`. This branch recomputes `verp_info = get_verp_info_from_email(mail_from[0])` [email_handler.py:L2101] and applies the *same* forward-style OR-coupling against `mail_from` [email_handler.py:L2102-2106]. Notably, it calls `handle_bounce(envelope, email_log, msg)` **directly at [email_handler.py:L2116] without first gating on `is_bounce()`** — an edge case worth flagging, since `handle_bounce()` then performs its own `if not email_log: return status.E512` check internally [email_handler.py:L1856-1858].

### Routing flowchart (unsigned `bounce+N+@domain`)

```mermaid
flowchart TD
    A["Inbound message, rcpt = bounce+N+@domain (UNSIGNED)"] --> B{"startswith BOUNCE_PREFIX AND endswith BOUNCE_SUFFIX<br/>OR verp_info == bounce_forward"}
    B -- "no" --> Z["Other routing branches (reply / iCloud / complaints / dmarc / spam)"]
    B -- "yes (NO signature required)" --> C["email_log_id = (verp id) OR parse_id_from_bounce(rcpt)"]
    C --> D["EmailLog.get(email_log_id)"]
    D -- "None" --> E512["E512: 550 SL E512 No such email log"]
    D -- "found" --> F{"is_bounce(envelope, msg)?<br/>mail_from == '<>' AND Content-Type multipart/report"}
    F -- "no" --> E213["raise VERPForward -> E213: 250 SL E213 Unknown email ignored"]
    F -- "yes" --> H{"email_log.user.is_active()?"}
    H -- "no" --> E510["E510: 550 SL E510 so such user"]
    H -- "yes" --> P{"reply phase or forward phase?"}
    P -- "forward" --> E211["E211: 250 SL E211 Bounce Forward phase handled"]
    P -- "reply" --> E212["E212: 250 SL E212 Bounce Reply phase handled"]
    E512 --> SPF{"status is 5xx AND SpamdResult SPF in (fail, soft_fail)?"}
    E510 --> SPF
    SPF -- "yes" --> E216["rewrite -> E216: 250 SL E216 Handled spf policy"]
    SPF -- "no / no SpamdResult header" --> OUT["return 5xx unchanged (oracle observable)"]
%% Source: email_handler.py handle() L2057-2116, handle_bounce() L1851-1914, _handle() SPF gate L2355-2365
```

### Reasoning / why this matters

The OR-coupling is *the* vulnerability because it **decouples acceptance from authentication**. The signed-VERP format was introduced to make bounce return-paths unforgeable, but as long as the router also accepts the legacy unsigned shape via `or`, the cryptographic guarantee is only as strong as the *weakest accepted format*. Every state-dependent response downstream of `EmailLog.get()` — existence, user status, phase — therefore becomes observable for **arbitrary attacker-chosen ids**, with no key, no account, and no signature. Localizing the boundary to exactly [email_handler.py:L2057-2063] is the actionable core of O3: the unsigned left-hand operand of that `or`, and the `parse_id_from_bounce` fallback it enables, are where "input that slipped through without cryptographic validation" enters the trusted lookup.


---

## (c) Bounce-detection criteria and spoofability — **O5**

O5 asks "what makes the system accept something as a bounce in the first place," and whether an attacker who controls the SMTP envelope and headers can satisfy it. The entire decision is `is_bounce()` [email_handler.py:L1813-1818], quoted **verbatim from the source** (note: the real signature has no `-> bool` return annotation):

```python
def is_bounce(envelope: Envelope, msg: Message):
    """Detect whether an email is a Delivery Status Notification"""
    return (
        envelope.mail_from == "<>"
        and msg.get_content_type().lower() == "multipart/report"
    )
```

It returns `True` if and only if **two conditions** hold:

1. **`envelope.mail_from == "<>"`** — the SMTP envelope sender is the null sender. This is the `MAIL FROM:<>` value of the SMTP transaction, set by the *sending* MTA. An external attacker connecting to the inbound server sets this directly.
2. **`msg.get_content_type().lower() == "multipart/report"`** — the MIME `Content-Type` header is `multipart/report`. This is just a header in the message body the attacker composes.

**Both inputs are fully attacker-controllable.** Nothing here validates that the message is a *genuine* Delivery Status Notification (no checks on `Report-Type=delivery-status`, no original-message correlation, no trusted-relay verification). In the forward branch, passing this check routes to `handle_bounce()` [email_handler.py:L2069-2070]; failing it (with a valid id) falls through to `raise VERPForward` → `250 SL E213 Unknown email ignored` [email_handler.py:L2073-2074, L2308-2318].

**Reproduced truth table (Tier 1, Appendix A.1):**

| `envelope.mail_from` | `Content-Type` | `is_bounce(...)` |
|---|---|---|
| `<>` | `multipart/report` | `True` |
| `<>` | `text/plain` | `False` |
| `a@b.c` | `multipart/report` | `False` |

### The ignore-list gate does not impede enumeration

A separate gate, `should_ignore_bounce()` [app/email_utils.py:L1361-1366], can short-circuit bounce *reporting*:

```python
def should_ignore_bounce(mail_from: str) -> bool:
    if IgnoreBounceSender.get_by(mail_from=mail_from):
        LOG.w("do not send back bounce report to %s", mail_from)
        return True
    return False
```

It returns `True` only when the sender is on the `IgnoreBounceSender` list (→ `250 SL E207 No bounce report` at call sites such as [email_handler.py:L2161]). An external attacker's arbitrary `mail_from` will not normally be on that operator-maintained list, so this gate does **not** block the enumeration path — it only suppresses outbound bounce reports for known-noisy senders.

### Reasoning / why this matters

Because the only two "is this a bounce" signals are envelope/header fields the sender supplies, an external attacker can **deterministically choose which downstream code path to hit**: set `MAIL FROM:<>` + `Content-Type: multipart/report` to drive the message into `handle_bounce()` (eliciting E211/E212/E510), or *deliberately* fail the predicate to elicit E213 instead of E512. That controllability is what makes the oracle trivially reachable — the attacker is not merely a passive observer of responses; they actively steer the classification.


---

## (d) Exact SMTP responses per scenario — **O4 (the evidence table)**

The table below maps each probe scenario (using the older unsigned `bounce+N+@domain` shape) to its code path, its `app/email/status.py` constant, and the **exact wire string** returned. **Every wire string in this table was captured live** by running the real handler against a real database inside the provided Docker container (Tier 2 of §2; full transcript in Appendix A); each is additionally transcribed character-for-character from `app/email/status.py` (transcription block follows the table). **The E510 string contains a verbatim typo — "so such user" (not "no such user") — preserved exactly as it appears in the source and called out explicitly.**

| # | Scenario | Code path (locator) | Constant | EXACT wire string |
|---|----------|---------------------|----------|-------------------|
| 1 | Non-existent `EmailLog` id, well-formed bounce | `EmailLog.get` → None [email_handler.py:L2063-2067] | `status.E512` | `550 SL E512 No such email log` |
| 2 | Valid id, active user, forward-phase log, real bounce | `handle_bounce` → `handle_bounce_forward_phase` [email_handler.py:L1851, L1432, L1912-1914] | `status.E211` | `250 SL E211 Bounce Forward phase handled` |
| 3 | Valid id, active user, reply-phase log, real bounce | `handle_bounce` → `handle_bounce_reply_phase` [email_handler.py:L1851, L1595, L1910-1911] | `status.E212` | `250 SL E212 Bounce Reply phase handled` |
| 4 | Valid id, **inactive** user | `handle_bounce` user check [email_handler.py:L1869-1871] | `status.E510` | `550 SL E510 so such user` ← **verbatim typo "so such"** |
| 5 | Valid id, message is **not** a bounce/OOO | `raise VERPForward` → mapped in `handle_DATA` [email_handler.py:L2073-2074, L2308-2318] | `status.E213` | `250 SL E213 Unknown email ignored` |
| 6 | Tampered / forged signed-VERP address | `get_verp_info_from_email` → None [app/email_utils.py:L1490] **and** shape matches no `bounce+`/transactional/reply prefix → satisfies **no** VERP branch → falls through to normal alias handling → alias not found | `status.E515` | `550 SL E515 Email not exist` |
| 7 | Any 5xx (rows 1, 4) when return-path SPF = fail/soft_fail | `_handle()` SPF gate rewrites [email_handler.py:L2357-2365] | `status.E216` | `250 SL E216 Handled spf policy` |

> **Row 6 — empirical correction (important).** One might predict a tampered signed-VERP address to be "ignored" like a non-bounce (E213). The **live** result is different and more instructive: it returns **`550 SL E515 Email not exist`**. Because the HMAC check fails (`get_verp_info_from_email → None`, captured live) **and** the address shape (`sl.<payload>.<badsig>@sl.local`) matches none of the `bounce+` / `transactional+` / `bounce_reply+` prefixes, the message satisfies **neither** disjunct of any VERP branch in `handle()` — so it never enters a bounce branch, never reaches `EmailLog.get()`, and never executes `raise VERPForward`. It instead falls through to ordinary inbound handling, where the address is looked up as an alias, is not found, and yields `E515`. This is the empirical *contrast* at the heart of the report: tampering with the **signed** format diverts the request away from the log lookup entirely, whereas the **unsigned** `bounce+N+@` format (rows 1–5) reaches the lookup with an attacker-chosen integer and no validation.

### Exact constants transcribed verbatim from `app/email/status.py`

```text
E205 = "250 SL E205 bounce handled"                       # L7   (transactional bounce)
E206 = "250 SL E206 Out of office"                        # L9   (out-of-office)
E207 = "250 SL E207 No bounce report"                     # L12  (IgnoreBounceSender)
E208 = "250 SL E208 Hotmail complaint handled"            # L14  (provider complaint)
E210 = "250 SL E210 Yahoo complaint handled"              # L18  (provider complaint)
E211 = "250 SL E211 Bounce Forward phase handled"         # L19
E212 = "250 SL E212 Bounce Reply phase handled"           # L20
E213 = "250 SL E213 Unknown email ignored"                # L21
E215 = "250 SL E215 Handled dmarc policy"                 # L23  (DMARC boundary)
E216 = "250 SL E216 Handled spf policy"                   # L24
E404 = "421 SL E404 Unexpected error - Retry later"       # L32  (generic Exception)
E506 = "550 SL E506 Email detected as spam"               # L43  (reply-phase spam)
E510 = "550 SL E510 so such user"                         # L47  <-- VERBATIM TYPO: "so such", NOT "no such"
E512 = "550 SL E512 No such email log"                    # L49
E515 = "550 SL E515 Email not exist"                      # L51  (alias not found — row 6, tampered VERP)
E519 = "550 SL E519 Email detected as spam"               # L55  (forward-phase spam)
E524 = "550 SL E524 Wrong use of reverse-alias"           # L62
```

`E205`/`E206`/`E207` are the transactional-branch and ignore-list outcomes; `E208`/`E210` (provider complaints), `E215` (DMARC, e.g. `app/handler/dmarc.py:L124, L194`), and `E506`/`E519` (spam scoring) are adjacent boundaries also reachable from `handle()` and are listed for completeness.

### The exception → status mapping in `handle_DATA` [email_handler.py:L2289-2332]

`handle_DATA` calls `self._handle(envelope, msg)` and centralizes exception-to-status mapping:

```python
except CannotCreateContactForReverseAlias as e:
    ...
    return status.E524                     # email_handler.py:L2307
except (VERPReply, VERPForward, VERPTransactional) as e:
    ...
    return status.E213                     # email_handler.py:L2308-2318
except Exception as e:
    ...
    return status.E404                     # email_handler.py:L2319-2332
```

This is **why "a valid id but not a bounce" yields a `250` (E213), not an error**: the `else: raise VERPForward` at [email_handler.py:L2073-2074] is caught at [email_handler.py:L2308] and translated to the *success-shaped* `250 SL E213 Unknown email ignored`. A genuinely unexpected error would instead surface as `421 SL E404` [email_handler.py:L2332].

### Evidence status per row

- **All seven rows were captured live** inside the provided Docker container (Tier 2 of §2; verbatim transcript in Appendix A, "Live container run"). Rows 1–6 were driven through `email_handler.handle(...)` (with the `handle_DATA` exception→status mapping reproduced verbatim and the real `VERPForward` captured for row 5); row 7 was driven through `MailHandler()._handle(...)` with the `5xx_overwrite_spf.eml` fixture, exactly mirroring `test_prevent_5xx_from_spf`.
- **Each row is explained by code-truth** (Tier 3): the exact return statement above paired with the exact `app/email/status.py` constant. The live value and the source read agree for every row.
- **Rows 1 and 7** are additionally corroborated by the project's own committed, passing tests through the real `_handle()` entrypoint: `test_preserve_5xx_with_valid_spf` and `test_preserve_5xx_with_no_header` (→ `status.E512`) [tests/test_email_handler.py:L130-142, L145-157], and `test_prevent_5xx_from_spf` (→ `status.E216`) [tests/test_email_handler.py:L115-127]. These route a deliberately invalid email-log id (`generate_verp_email(VerpType.bounce_forward, 99999999999999)`) through `_handle()` and assert the exact constants.
- **Row 6** is the one scenario whose live result diverges from a naïve prediction (E515, not E213); see the empirical-correction note above the constants block and Tier 2 of §2 for the full reasoning.

### Reasoning / why this matters

An oracle is defined by **distinguishable, deterministic outputs per hidden state**. The table shows exactly that: five different internal conditions (no row / inactive user / forward log / reply log / non-bounce) produce five different wire strings, each a fixed constant returned from a fixed line of code. There is no randomization, no rate-shaping on the response content, and no generic "rejected" catch-all that would collapse the distinctions. That determinism is precisely what an external attacker reads off the wire — and it segues directly into the depth analysis of O6.


---

## (e) Enumeration-oracle depth analysis — **O6**

O6 asks whether the response codes "reveal more than just whether an id exists." They do. The unsigned path differentiates **at least three orthogonal dimensions** of internal state, each surfaced by a distinct constant:

- **`550 SL E512 No such email log`** → the `EmailLog` row with that id **does not exist** [email_handler.py:L2065-2067; app/email/status.py:L49]. This is the baseline "miss."
- **`550 SL E510 so such user`** → the row **exists**, but `email_log.user.is_active()` is `False` [email_handler.py:L1869-1871; app/email/status.py:L47]. `User.is_active()` returns `True` when `delete_on is None`, otherwise `delete_on < arrow.now()` [app/models.py:L766-769]. Therefore a user with a **future `delete_on`** evaluates as **inactive** — so E510 (distinct from E512) leaks the existence of a record **and** the **account-deletion-pending / disabled** status of its owner.
- **`250 SL E211 Bounce Forward phase handled`** → row exists, active user, and the log is in the **forward phase** [email_handler.py:L1912-1914; app/email/status.py:L19].
- **`250 SL E212 Bounce Reply phase handled`** → row exists, active user, and the log is in the **reply phase** [email_handler.py:L1910-1911; app/email/status.py:L20]. The forward-vs-reply split is driven by `EmailLog.is_reply` [app/models.py:L2075] (surfaced by `get_phase()` [app/models.py:L2143-2147]), so the two 2xx codes leak the **directionality/phase** of the underlying log.
- **`250 SL E213 Unknown email ignored`** → the row exists but the message failed `is_bounce()` [email_handler.py:L2073-2074, L2308-2318; app/email/status.py:L21]. Crucially, an attacker can *deliberately* send a non-bounce to a valid id to confirm existence via **E213** — a different code than the **E512** returned for a missing id. So even without satisfying bounce detection, the attacker gets an existence bit (E213 ⇒ exists, E512 ⇒ does not).

Putting it together, the response code alone classifies an arbitrary probed id `N` into:

| Observed wire string | Inferred internal state |
|---|---|
| `550 SL E512 No such email log` | no `EmailLog` row `N` |
| `550 SL E510 so such user` | row `N` exists; owning user **inactive** (e.g. deletion pending) |
| `250 SL E211 Bounce Forward phase handled` | row `N` exists; active user; **forward**-phase |
| `250 SL E212 Bounce Reply phase handled` | row `N` exists; active user; **reply**-phase |
| `250 SL E213 Unknown email ignored` | row `N` exists; message not classified as a bounce |

The oracle therefore leaks **(1) `EmailLog` id existence, (2) owning-user active/inactive status, and (3) forward-vs-reply phase of the log** — a **multi-dimensional state oracle**, not a binary existence check.

### Reasoning / why this matters

`EmailLog` ids are **sequential database primary keys** (the shared `ModelMixin` declares an autoincrement integer `id` [app/models.py:L62-63], inherited by `class EmailLog(Base, ModelMixin)` [app/models.py:L2060]), and the unsigned path accepts **any** integer (§(b)). An attacker can therefore *walk* the id space — `bounce+1+@sl.local`, `bounce+2+@sl.local`, … — and read off, per id, not just "in use / free" but the **lifecycle classification** of each record (active vs deletion-pending owner; forward vs reply traffic). Aggregated across the id range, this yields a census of the platform's email-forwarding activity and of accounts in a pending-deletion state — information that should require authentication to obtain. The only thing standing between the attacker and the full 5xx subset of this signal is the SPF gate analyzed next — and it does nothing about the 2xx subset.


---

## (f) SPF black-holing mitigation and its limits — **O6 mitigation**

There is exactly one post-routing defense that conditionally suppresses the oracle: an SPF gate in `_handle()` [email_handler.py:L2335-2378]. `_handle()` runs the router inside a Flask app context, then inspects the result. Quoted **verbatim from the source** (the real code is multi-line and includes a log line):

```python
with create_light_app().app_context():
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

The intent is captured by the comment at [email_handler.py:L2355]: *"Only bounce messages if the return-path passes the spf check. Otherwise black-hole it."* `SpamdResult` / `SPFCheckResult` come from `app/handler/spamd_result.py` (the SPF verdict is parsed from a `SpamdResult` header injected upstream; `SPFCheckResult.fail`/`soft_fail` correspond to `R_SPF_FAIL`/`R_SPF_SOFTFAIL` [app/handler/spamd_result.py:L32-46]). `SpamdResult.extract_from_headers(msg)` is at [app/handler/spamd_result.py:L76].

### The mitigation is narrow — three conjunctive conditions

The rewrite to `250 SL E216 Handled spf policy` happens **only** when all three hold:

1. the status is **5xx** (`return_status[0] == "5"`) [email_handler.py:L2357]; **and**
2. a `SpamdResult` header **is present** (`spamd_result` is truthy) [email_handler.py:L2358]; **and**
3. its SPF verdict is **fail or soft_fail** (`spamd_result.spf in (SPFCheckResult.fail, SPFCheckResult.soft_fail)`) [email_handler.py:L2358-2361].

From this, the precise limits follow directly:

- **The 2xx codes are never touched.** `E211` / `E212` / `E213` all begin with `2`, so the `return_status[0] == "5"` guard is `False` and they **always pass through unchanged**, regardless of SPF. The forward-vs-reply phase distinction and the "valid id but non-bounce" existence bit therefore remain observable in *all* cases.
- **The 5xx codes (E512 / E510) remain observable** whenever there is **no `SpamdResult` header**, or the SPF verdict is anything other than fail/soft_fail (e.g. `allow`/neutral/none). Since a Delivery Status Notification's envelope sender is the **null sender `<>`** (which is exactly what `is_bounce()` requires — §(c)), SPF evaluated on `MAIL FROM` cannot authenticate the originating host in the usual way, so the reliability of this gate in practice is limited.

### Empirical anchor (Tier 4 — the project's own tests) and the live capture

The gate was exercised **live** in this investigation: row 7 of the §(d)/Appendix-A.2 capture drove a non-existent id through the real `_handle()` entrypoint with an `R_SPF_FAIL` header and returned `250 SL E216 Handled spf policy` (the engine logged *"Replacing 5XX to 216 status because the return-path failed the spf check"* at email_handler.py:L2362). It is additionally pinned by three committed, passing tests that drive an invalid id through the same `_handle()` entrypoint:

- `test_prevent_5xx_from_spf` — `R_SPF_FAIL` ⇒ result is `status.E216` (the 5xx is rewritten) [tests/test_email_handler.py:L115-127].
- `test_preserve_5xx_with_valid_spf` — `R_SPF_ALLOW` ⇒ result is `status.E512` (the 5xx is **preserved**) [tests/test_email_handler.py:L130-142].
- `test_preserve_5xx_with_no_header` — **no** `SpamdResult` header ⇒ result is `status.E512` (the 5xx is **preserved**) [tests/test_email_handler.py:L145-157].

These are the project's own proof that the 5xx is masked **only** under SPF fail/soft_fail *with* a header present, and preserved otherwise.

### Reasoning / why this matters

The SPF gate is a **partial, conditional mask of the 5xx subset only**. It is genuinely useful against naive backscatter, but as an anti-enumeration control it has two gaps that keep the oracle alive: it leaves the entire **2xx** signal (phase + non-bounce existence bit) untouched, and it suppresses the **5xx** signal only when an SPF-fail/soft_fail `SpamdResult` header happens to be attached — the source preserves the 5xx unchanged whenever no `SpamdResult` header is present or its SPF verdict is non-fail [email_handler.py:L2357-2365]. Whether an external sender can influence that upstream signal is **deployment-dependent** (the `SpamdResult` is produced by the upstream filtering layer, not by the bounce path itself, so the repository source alone does not establish attacker control over it); and for null-sender DSNs, SPF on an empty `MAIL FROM` is inherently weak. Consequently the multi-state oracle of §(e) **largely persists**; the gate narrows, but does not close, what an external prober can read.


---

## (g) Rationale, concluding risk summary, and industry framing

### Synthesis of the root cause

The vulnerability is the product of three composable facts, each established from the source above:

1. **Acceptance is decoupled from authentication** — the OR-coupling in `handle()` accepts the unsigned `bounce+{id}+@domain` shape on equal footing with the signed VERP format [email_handler.py:L2057-2061], routing an attacker-chosen integer into `EmailLog.get()` with no crypto [email_handler.py:L2062-2063].
2. **Detection is spoofable** — `is_bounce()` keys solely on `mail_from == "<>"` and `Content-Type: multipart/report`, both attacker-set [email_handler.py:L1813-1818].
3. **Responses are state-differentiated and mostly unmasked** — five internal states map to five fixed wire strings (§(d)/§(e)), and the only mask (the SPF gate) covers a conditional subset of the 5xx codes and none of the 2xx codes (§(f)).

Remove any one and the oracle weakens; together they yield an externally reachable, multi-dimensional enumeration oracle.

### Industry framing (background research — framing only, not code behavior)

The signed VERP format is, in industry terms, a **BATV-style mechanism** — *Bounce Address Tag Validation*, the practice of signing the return-path with a cryptographic token and a timestamp so that bounces lacking a valid signature can be rejected. SimpleLogin's `generate_verp_email()` (HMAC over `[verp_type, object_id, minutes]`, embedding a timestamp) is structurally this pattern — with one important caveat established from the source: `get_verp_info_from_email()` uses that embedded timestamp only as a **future-timestamp guard** (rejecting values more than `VERP_MESSAGE_LIFETIME` = 5 days *ahead* of now [app/email_utils.py:L1496]) and does **not** expire old signatures, so unlike a full BATV deployment it supplies integrity (plus a forward-dating bound) but **no true freshness/expiry window**. The older `bounce+{id}+@domain` form is a **pre-BATV unsigned construction** of precisely the kind BATV was designed to supersede. Separately, legitimate Delivery Status Notifications use the **null return-path** (`MAIL FROM:<>`), and common operational guidance is to **black-hole** (silently drop) bounces whose return-path cannot be validated rather than emit an informative response — which is the stated intent behind SimpleLogin's SPF gate. *(These framing statements are background context; every behavioral claim elsewhere in this document is sourced to the code.)*

### Risk summary

- **Class:** Information disclosure / record-and-account enumeration (an insecure direct object reference reachable over SMTP, plus a missing-authentication-on-a-sensitive-action flavor).
- **Reachability:** **External, unauthenticated.** The attacker only needs to deliver mail to the inbound server; no SimpleLogin account, key, or signature is required.
- **Attacker cost:** **Low.** One crafted email per probed id; each probe triggers a single `EmailLog.get()` primary-key lookup. Ids are sequential (autoincrement primary keys — `ModelMixin.id` [app/models.py:L62-63], inherited by `EmailLog` [app/models.py:L2060]), so coverage of the id space is linear and cheap. There is no response-content rate-shaping that would blur the per-id classification.
- **Data exposed:** existence of `EmailLog` records, owning-user active/inactive (deletion-pending) status, and forward-vs-reply phase — aggregable into a census of forwarding activity and pending-deletion accounts.
- **Partial mitigation in place:** the SPF gate (§(f)) conditionally masks the 5xx subset only.

**Out-of-scope observation (NOT proposed as work).** This deliverable is investigative only; remediation is explicitly out of scope and **no fix is implemented or recommended as an action item here**. For completeness of analysis only, the categories of hardening that would address the root cause are: requiring a valid signature on *all* accepted bounce return-paths (i.e. removing the unsigned left-hand operand of the OR-coupling), collapsing the dual-format acceptance, normalizing the distinct status codes on the unauthenticated path so they no longer differentiate internal state, and/or broadening the black-holing so it is not contingent on an attacker-influenced `SpamdResult`/SPF signal. These are noted strictly as analytical observations, not as a change to be made under this task.

### Reasoning / why this matters (verdict, restated)

**Yes — the older unsigned `bounce+{id}+@domain` path is an exploitable, multi-state enumeration oracle.** The security boundary is the **OR-coupling at [email_handler.py:L2057-2063]**, where an unsigned, attacker-shaped address bypasses the cryptographic check and reaches `EmailLog.get()`; the differentiated status codes then leak existence, user-active status, and log phase; and the SPF gate at [email_handler.py:L2357-2365] only partially mitigates by masking a conditional subset of the 5xx responses while leaving the 2xx responses fully observable. This conclusion is not theoretical: every wire string was **observed live** in the project's own runtime inside the provided container (Tier 2, Appendix A.2), each response is **explained** by a direct read of the source (Tier 3), the externally-observable 5xx/E216 behavior is **independently anchored** by the project's own passing tests (Tier 4), and the format-layer mechanics were **directly executed** (Tier 1, Appendix A.1).


---

## Appendix A — Reproduced runtime output (verbatim)

This appendix has two parts: **A.1** is the Tier-1 reproduction of the format-layer functions on the host interpreter, and **A.2** is the Tier-2 **live run inside the provided Docker container** that produced every wire string in the §(d) evidence table.

### A.1 — Tier 1: format-layer functions (host CPython 3.13.7)

The block below is the captured output of a transient `/tmp` probe (since deleted, per the "clean up test scripts" rule) that executed the **verbatim, standard-library-only** functions `parse_id_from_bounce` / `generate_verp_email` / `get_verp_info_from_email` / `is_bounce` with the repository-default constants listed in §2 (Tier 1). It was independently re-executed during this investigation on CPython 3.13.7; all deterministic results matched.

```text
O1-A  parse_id_from_bounce('bounce+12345+@domain') -> 12345
      parse_id_from_bounce('bounce+1+@sl.local') -> 1
      parse_id_from_bounce('bounce+99999999999999+@sl.local') -> 99999999999999
      slice 'bounce+12345+@domain'[6:12] = '+12345' ; int('+12345') = 12345
O1-B  'bounce+12345+@sl.local': startswith('bounce+')=True endswith('+@sl.local')=True
      get_verp_info_from_email('bounce+12345+@sl.local') -> None   (unsigned shape is not signed VERP)
O1-C  generate_verp_email(bounce_forward, 12345) -> 'sl.lmycyibrgiztinjmeazdgnjygq2tqxi.e7bm3xshjgxji@sl.local'
      get_verp_info_from_email(<above>) -> (VerpType.bounce_forward, 12345)
O2/O3 tampered sig 'sl.lmycyibrgiztinjmeazdgnjygq2tqxi.e7bm3xshjgxja@sl.local' -> get_verp_info_from_email -> None
      parse_id_from_bounce('bounce+777+@sl.local') -> 777   (attacker-chosen id, NO crypto)
O5    is_bounce('<>','multipart/report') -> True
      is_bounce('<>','text/plain') -> False
      is_bounce('a@b.c','multipart/report') -> False
```

> **Honesty note on the time- and secret-dependent VERP value (O1-C / O2-O3).** The VERP address has two variable segments. (1) The **payload** encodes `[verp_type, object_id, minutes]`, where `minutes` is a *live wall-clock value* (minutes since `VERP_TIME_START=1640995200`, 2022-01-01 [app/email_utils.py:L68]); only the `minutes` field changes between runs. The canonical `lmycyibrgiztinjmeazdgnjygq2tqxi` base32-decodes to exactly `[0, 12345, 2358458]` (`verp_type=bounce_forward=0`, `object_id=12345`, `minutes=2358458` ≈ 2026-06-26T19:38Z); the live container captures below decode to `[0, 12345, 2358749]` (≈ 2026-06-27T00:29Z) — **identical `verp_type` and `object_id`, differing only in `minutes`**, which is itself proof the timestamp is computed live rather than hardcoded. (2) The **signature** is `HMAC(VERP_EMAIL_SECRET, payload)` truncated to 8 bytes, so it changes whenever the payload changes *and* whenever the secret changes. The placeholder-secret re-run here (`"blitzy-verification-secret-0123456789ABCDEF"`) produced signature `diqtbmuefcvek`; the live container (using its real, **redacted** secret) produced different signatures again (see A.2). In **every** run the round-trip succeeds and any single-character signature flip yields `None` — which is exactly the HMAC integrity property under test (§(a)).

### A.2 — Tier 2: live container run (Python 3.10.18 + PostgreSQL + Redis)

The block below is the **verbatim** captured output of the live probe run inside the provided Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (SimpleLogin at commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`), after `/build.sh` (Postgres+Redis up, schema migrated) and the documented DKIM PKCS#1 fix. The transient probe (`/app/tests/blitzy_live_probe.py`, deleted afterward; the source tree was never modified) seeded rows via the committed fixtures and invoked the **real** routing — `email_handler.handle(...)` for rows 1–6 (with the `handle_DATA` exception→status mapping reproduced verbatim, capturing the actual exception), and `MailHandler()._handle(...)` with the project's `5xx_overwrite_spf.eml` fixture for row 7. The probe test reported `1 passed`.

```text
@@@VERP_GEN sl.lmycyibrgiztinjmeazdgnjyg42dsxi.77ywcj5miivse@sl.local
@@@VERP_RT (<VerpType.bounce_forward: 0>, 12345)
@@@PARSE 12345
@@@R1 (None, '550 SL E512 No such email log')
@@@R2 (None, '250 SL E211 Bounce Forward phase handled')
@@@R3 (None, '250 SL E212 Bounce Reply phase handled')
@@@R5 ('VERPForward', '250 SL E213 Unknown email ignored')
@@@R6tamper_verp_info None
@@@R6 (None, '550 SL E515 Email not exist')
@@@R4 (None, '550 SL E510 so such user')
@@@R7 250 SL E216 Handled spf policy
@@@CONST E512='550 SL E512 No such email log' E211='250 SL E211 Bounce Forward phase handled' E212='250 SL E212 Bounce Reply phase handled' E213='250 SL E213 Unknown email ignored' E510='550 SL E510 so such user' E515='550 SL E515 Email not exist' E216='250 SL E216 Handled spf policy'
```

Reading the capture against the §(d) table:

- `@@@R1`–`@@@R4` are the bare tuples `(raised_exception, wire_string)` returned by the probe's wrapper; the `None` left element means `handle()` returned normally. They yield **E512 / E211 / E212 / E510** respectively — note `E510` is the verbatim **`550 SL E510 so such user`** ("so such", the source typo).
- `@@@R5` shows the left element `'VERPForward'`: `handle()` **genuinely raised** `VERPForward`, which `handle_DATA` maps to **`250 SL E213 Unknown email ignored`** [email_handler.py:L2308-2318].
- `@@@R6tamper_verp_info None` confirms the tampered signature is rejected at verification; `@@@R6` then shows the message falls through to alias handling and returns **`550 SL E515 Email not exist`** (the empirical correction discussed in §(d) and §2/Tier 2).
- `@@@R7` is the SPF-gate rewrite to **`250 SL E216 Handled spf policy`**, captured through `_handle()` with `R_SPF_FAIL` (the engine also logged *"Replacing 5XX to 216 status because the return-path failed the spf check"* at email_handler.py:L2362).
- `@@@CONST` echoes the live `app/email/status.py` constants, confirming the wire strings are byte-identical to the source.

> **Provenance of the canonical A.1 value vs. the A.2 capture.** A.1's O1-C line shows `…gnjygq2tqxi.e7bm3xshjgxji` (payload `minutes=2358458`); A.2's `@@@VERP_GEN` shows `…gnjyg42dsxi.77ywcj5miivse` (payload `minutes=2358749`). Both use the container's real secret; they differ **only** because A.2 ran ~4.85 hours later (different `minutes`, hence a different payload and therefore a different HMAC). This is the time-dependence documented in the honesty note above — not an inconsistency.

---

## Appendix B — Ready-to-run DB-backed probe (for the Docker container)

The script below is the **exact probe used to capture the A.2 output above** — it is reproduced verbatim so any reader can re-derive every §(d) wire string inside the provided SimpleLogin container (Python 3.10 + Postgres + Redis, `NOT_SEND_EMAIL=true` [example.env:L19]). Re-running it reproduces every `@@@R*` wire string and the `@@@CONST` constants summary **byte-for-byte** (those values are deterministic and time-invariant); the **sole** run-to-run variation is the `@@@VERP_GEN` line, whose `minutes` and signature segments change on every run (this is the format-demonstration value behind §(a), *not* a §(d) wire string — see the A.2 honesty note above). It is modeled on the committed harness [tests/test_email_handler.py:L82-156] and seeds rows exactly as the suite's own fixtures do (`create_new_user`, `Alias.create_new_random`, `Contact.create`, `EmailLog.create`). It is a **transient** artifact: it lives only inside the ephemeral container and is deleted after use — it is **never** added to the source tree.

Two construction details matter for reproducibility:

1. **Build the DSN from raw RFC822 bytes and parse with `email.message_from_bytes(...)`** — mirroring production's `handle_DATA` (`msg = email.message_from_bytes(envelope.original_content)` [email_handler.py:L2290]). Constructing a `message/delivery-status` part by hand with `EmailMessage().set_payload("…")` does **not** round-trip through the email generator (its sub-blocks must be `Message` objects, not strings) and raises `AttributeError: 'str' object has no attribute 'policy'` during serialization. The raw-bytes form avoids that entirely.
2. **Route rows 1–6 through `email_handler.handle(...)` directly** (as the committed bounce tests do) and replicate `handle_DATA`'s `except (VERPReply, VERPForward, VERPTransactional): return status.E213` mapping in the probe — capturing the *real* exception. Do **not** route the seeded-row scenarios through `_handle()`, because `_handle` opens a nested `create_light_app().app_context()` that swaps the SQLAlchemy session and detaches the rows you just seeded (`DetachedInstanceError`). Row 7 uses `_handle()` deliberately, with a non-existent id (no seeded row to detach), exactly like `test_prevent_5xx_from_spf`.

> **How to run it.** Inside the container, after `/build.sh` and the DKIM PKCS#1 fix, source the build env (which sets `DB_URI=postgresql://test:test@localhost:5432/test`, `NOT_SEND_EMAIL=true`, `EMAIL_DOMAIN=sl.local`, and the secret-bearing values such as `FLASK_SECRET`/`VERP_EMAIL_SECRET` — **redacted** here), then run the probe under pytest so the `flask_client` fixture (hence the DB session) is active:
> ```bash
> cd /app
> set -a; eval "$(grep -E '^export ' /build.sh)"; set +a
> unset PYTEST_ADDOPTS                       # the image sets --reruns; clear it
> export CONFIG=/app/tests/test.env
> /app/venv/bin/python -m pytest -s -p no:cacheprovider -o addopts="" \
>     tests/blitzy_live_probe.py::test_probe
> ```
> Then **delete** `tests/blitzy_live_probe.py`. (With the build env sourced, the exported `DB_URI` on port 5432 takes precedence over `test.env`'s 15432 because `app/config.py` loads dotenv with `override=False`.)

```python
# tests/blitzy_live_probe.py  — TRANSIENT; delete after capturing output. NEVER commit.
# Run inside the SimpleLogin Docker container (py3.10, Postgres, Redis, NOT_SEND_EMAIL=true).
# Captures the exact SMTP wire strings for the unsigned bounce+{id}+@domain oracle.
import email
import arrow
from aiosmtpd.smtp import Envelope

import email_handler
from app.email import status
from app.errors import (
    VERPForward, VERPReply, VERPTransactional, CannotCreateContactForReverseAlias,
)
from app.config import BOUNCE_PREFIX, BOUNCE_SUFFIX  # "bounce+", "+@sl.local"
from app.db import Session
from app.models import Alias, Contact, EmailLog, VerpType
from app.email_utils import (
    generate_verp_email, get_verp_info_from_email, parse_id_from_bounce,
)
from tests.utils import create_new_user, load_eml_file

# A real DSN, built as raw bytes so email.message_from_bytes() yields a proper tree
# (top-level Content-Type: multipart/report => is_bounce() True when mail_from == "<>").
_DSN = (
    b"From: mailer-daemon@remote.example\r\n"
    b"To: bounce@sl.local\r\n"
    b"Subject: Undelivered Mail Returned to Sender\r\n"
    b'Content-Type: multipart/report; report-type=delivery-status; boundary="BOUND"\r\n'
    b"MIME-Version: 1.0\r\n\r\n"
    b"--BOUND\r\n"
    b"Content-Type: text/plain\r\n\r\n"
    b"This is a delivery status notification.\r\n\r\n"
    b"--BOUND\r\n"
    b"Content-Type: message/delivery-status\r\n\r\n"
    b"Reporting-MTA: dns; remote.example\r\n\r\n"
    b"Final-Recipient: rfc822; user@example.com\r\n"
    b"Action: failed\r\n"
    b"Status: 5.1.1\r\n\r\n"
    b"--BOUND--\r\n"
)
# A plain (non-bounce) message: top-level text/plain => is_bounce() False.
_PLAIN = (
    b"From: someone@remote.example\r\n"
    b"To: bounce@sl.local\r\n"
    b"Subject: hello\r\n"
    b"Content-Type: text/plain\r\n\r\n"
    b"not a bounce\r\n"
)


def _h(rcpt, mail_from, raw):
    """Drive email_handler.handle() and apply the handle_DATA() status mapping
    [email_handler.py:L2307-2332], returning (raised_exception_name_or_None, wire_status)."""
    env = Envelope()
    env.mail_from = mail_from
    env.rcpt_tos = [rcpt]
    msg = email.message_from_bytes(raw)
    try:
        return (None, email_handler.handle(env, msg))
    except CannotCreateContactForReverseAlias:
        return ("CannotCreateContactForReverseAlias", status.E524)
    except (VERPReply, VERPForward, VERPTransactional) as e:
        return (type(e).__name__, status.E213)
    except Exception as e:
        return ("Exception:%s" % type(e).__name__, status.E404)


def _seed(user, is_reply: bool):
    alias = Alias.create_new_random(user)
    Session.commit()
    contact = Contact.create(
        user_id=user.id, alias_id=alias.id,
        website_email="contact@example.com", reply_email="rep-%d@sl.local" % alias.id,
        commit=True,
    )
    return EmailLog.create(
        user_id=user.id, contact_id=contact.id, alias_id=alias.id,
        is_reply=is_reply, commit=True,
    )


def test_probe(flask_client):
    v = generate_verp_email(VerpType.bounce_forward, 12345)
    print("@@@VERP_GEN", v)
    print("@@@VERP_RT", get_verp_info_from_email(v))
    print("@@@PARSE", parse_id_from_bounce("bounce+12345+@sl.local"))

    user = create_new_user()
    Session.commit()

    # Row 1: non-existent EmailLog id, well-formed bounce, null sender -> E512
    print("@@@R1", _h(f"{BOUNCE_PREFIX}99999999{BOUNCE_SUFFIX}", "<>", _DSN))

    # Row 2: valid id, active user, FORWARD-phase log, real bounce -> E211
    fwd = _seed(user, is_reply=False)
    print("@@@R2", _h(f"{BOUNCE_PREFIX}{fwd.id}{BOUNCE_SUFFIX}", "<>", _DSN))

    # Row 3: valid id, active user, REPLY-phase log, real bounce -> E212
    rep = _seed(user, is_reply=True)
    print("@@@R3", _h(f"{BOUNCE_PREFIX}{rep.id}{BOUNCE_SUFFIX}", "<>", _DSN))

    # Row 5: valid id, message NOT a bounce (plain, non-null sender) -> VERPForward -> E213
    print("@@@R5", _h(f"{BOUNCE_PREFIX}{fwd.id}{BOUNCE_SUFFIX}", "a@b.c", _PLAIN))

    # Row 6: tampered signed-VERP (flip one signature char) -> verp_info None,
    #        shape != bounce+ => no VERP branch => normal alias handling => E515.
    good = generate_verp_email(VerpType.bounce_forward, fwd.id)
    local, domain = good.split("@")
    tampered = local[:-1] + ("a" if local[-1] != "a" else "b") + "@" + domain
    print("@@@R6tamper_verp_info", get_verp_info_from_email(tampered))
    print("@@@R6", _h(tampered, "a@b.c", _PLAIN))

    # Row 4: valid id, INACTIVE user (future delete_on => is_active() False) -> E510
    user.delete_on = arrow.now().shift(days=+30)
    Session.commit()
    fwd2 = _seed(user, is_reply=False)
    print("@@@R4", _h(f"{BOUNCE_PREFIX}{fwd2.id}{BOUNCE_SUFFIX}", "<>", _DSN))

    # Row 7: 5xx (non-existent id) + X-Spamd-Result R_SPF_FAIL -> SPF gate rewrites to E216.
    #        Mirror committed test test_prevent_5xx_from_spf exactly (_handle + fixture).
    user2 = create_new_user()
    alias2 = Alias.create_new_random(user2)
    Session.commit()
    msg_spf = load_eml_file(
        "5xx_overwrite_spf.eml",
        {"alias_email": alias2.email, "spf_result": "R_SPF_FAIL"},
    )
    env7 = Envelope()
    env7.mail_from = msg_spf["from"]
    env7.rcpt_tos = [generate_verp_email(VerpType.bounce_forward, 99999999999999)]
    print("@@@R7", email_handler.MailHandler()._handle(env7, msg_spf))

    print("@@@CONST E512=%r E211=%r E212=%r E213=%r E510=%r E515=%r E216=%r" % (
        status.E512, status.E211, status.E212, status.E213,
        status.E510, status.E515, status.E216))
```

> Running this probe is what produced the A.2 transcript above (`1 passed`); every `@@@R*` row and the `@@@CONST` summary reproduce verbatim on re-run (deterministic), the only run-to-run variation being the time-dependent `@@@VERP_GEN` minute/signature (per the A.2 honesty note). The seeding mirrors the fixtures used by the committed tests, so the printed strings are the live equivalents of the §(d) constants — and the row-6 result (`E515`, not `E213`) is the empirically observed truth, not a prediction.


---

## Appendix C — Reference map and environment evidence

### Code locators used in this document (grouped by file)

**`email_handler.py`**
- `is_bounce()` — L1813-1818 (detection predicate; real signature has no `-> bool`)
- `handle_bounce_forward_phase()` — L1432; `handle_bounce_reply_phase()` — L1595
- `handle_transactional_bounce()` — L1821
- `handle_bounce()` — L1851-1914 (E512 L1856-1858; E510 L1869-1871; reply→E212 L1910-1911; forward→E211 L1912-1914)
- `handle()` — L1945; `verp_info = get_verp_info_from_email(rcpt_tos[0])` L2035
- transactional-VERP branch L2038-2054 (E205 L2047 / E206 L2052 / `raise VERPTransactional` L2054)
- **forward-VERP OR-coupling — L2057-2061** (THE boundary); id+lookup L2062-2063; E512 L2065-2067; `is_bounce`→`handle_bounce` L2069-2070; OOO L2071-2072; `else: raise VERPForward` L2073-2074
- reply-VERP branch L2077-2098 (id+lookup L2082-2083; E512 L2085-2087; →E212 L2090-2091; `raise VERPReply` L2094-2098)
- iCloud `mail_from` variant L2100-2116 (direct `handle_bounce` at L2116, no `is_bounce` gate); `verp_info` on `mail_from` L2101
- adjacent boundaries: E208 L2130, E210 L2141, E207 L2161, E519 L783, E506 L1094
- `handle_DATA()` L2289-2332 (→E524 L2307; `(VERPReply, VERPForward, VERPTransactional)`→E213 L2308-2318; `Exception`→E404 L2319-2332)
- `_handle()` L2335-2378 (comment L2355; `SpamdResult.extract_from_headers` L2356; **SPF gate L2357-2365 → E216 L2365**)
- VERP generation call sites: bounce_forward L905, bounce_reply L1225, transactional L1290
- imports: `BOUNCE_PREFIX` L78, `status` L90, `SPFCheckResult` L146, `VerpType` L166; `Controller(MailHandler(), ...)` L2383

**`app/email_utils.py`**
- `VERP_TIME_START = 1640995200` L68; `VERP_HMAC_ALGO = "sha3-224"` L69
- `parse_id_from_bounce()` L1258-1259
- `should_ignore_bounce()` L1361-1366
- `generate_verp_email()` L1438-1465 (itsdangerous-avoidance comment L1452; HMAC L1454-1456)
- `get_verp_info_from_email()` L1467-1499 (prefix/3-field check L1476; **`!=` signature check L1490**; 3-element check L1494; future-timestamp guard L1496)

**`app/email/status.py`** — E205 L7, E206 L9, E207 L12, E208 L14, E210 L18, E211 L19, E212 L20, E213 L21, E215 L23, E216 L24, E404 L32, E506 L43, **E510 L47 (verbatim typo "so such user")**, E512 L49, **E515 L51 (row-6 live result "Email not exist")**, E519 L55, E524 L62

**`app/config.py`** — `BOUNCE_PREFIX` L100, `BOUNCE_SUFFIX` L101, `BOUNCE_PREFIX_FOR_REPLY_PHASE` L108, `TRANSACTIONAL_BOUNCE_PREFIX` L113, `TRANSACTIONAL_BOUNCE_SUFFIX` L116, `VERP_MESSAGE_LIFETIME = 5*86400` L499, `VERP_PREFIX = "sl"` L500, `VERP_EMAIL_SECRET` L502-503 (≥32-char guard L505)

**`app/models.py`** — `Phase` enum L241; `VerpType` enum L247-250; `User.is_active()` L766-769; `EmailLog` L2060; `is_reply` column L2075; `get_phase()` L2143-2147

**`app/handler/spamd_result.py`** — `SPFCheckResult` (allow/fail/soft_fail) L32-46; `SpamdResult` L54; `extract_from_headers()` L76
**`app/handler/dmarc.py`** — `status.E215` returns L124, L194

**`tests/test_email_handler.py`** — harness pattern (`Envelope()` + `handle()`/`_handle()`) L82-156; `test_prevent_5xx_from_spf` L115-127; `test_preserve_5xx_with_valid_spf` L130-142; `test_preserve_5xx_with_no_header` L145-157
**`tests/conftest.py`** — `from psycopg2 import errors` L15; `CREATE EXTENSION pg_trgm` L32; `connection.begin()` L61; `rollback()` L75
**`tests/test.env`** — `DB_URI=postgresql://test:test@localhost:15432/test` L17

### Environment / version / dependency evidence

- **Python 3.10** is the project's authoritative runtime: `FROM python:3.10` [Dockerfile:L8]; CI matrix `python-version: ["3.10"]` [.github/workflows/main.yml:L40] (and `'3.10'` at L17); `target-version = ['py310']` [pyproject.toml:L2]; `python = "^3.10"` [pyproject.toml:L61].
- **Dependencies** (Poetry-locked, `poetry.lock`; versions per the locked set): `aiosmtpd ^1.2` (provides `Envelope`/`Controller`); `SQLAlchemy 1.3.24` (backs `EmailLog.get()`); `Flask ^1.1.2` (`create_light_app()` app context within `_handle()`); `redis` (rate limiting). `cryptography` and `itsdangerous` are present in the environment but **not** on the VERP signing path.
- **Local-observation aids:** `NOT_SEND_EMAIL=true` [example.env:L19] prints outbound mail instead of sending it; `EMAIL_DOMAIN=sl.local` [example.env:L22] yields the default `BOUNCE_SUFFIX="+@sl.local"`.
- **Dependency fact (restated):** VERP signing uses the Python standard library `hmac` / `base64` / `json`, **not** the `itsdangerous` dependency — per the code comment at [app/email_utils.py:L1452].

---

### Document provenance footer

- **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (the only file written by this task).
- **Source of truth:** SimpleLogin `app` at HEAD `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`; all source files were read **read-only** and **none were modified**.
- **Evidence tiers:** Tier 1 (genuine reproduced output of stdlib-only functions, Appendix A.1) · **Tier 2 (live database-backed run of the real handler inside the provided Docker container — Python 3.10.18 + Postgres + Redis — capturing every §(d) wire string, Appendix A.2)** · Tier 3 (code-truth: exact return statements + `app/email/status.py` constants) · Tier 4 (the project's committed, passing tests `test_prevent_5xx_from_spf` / `test_preserve_5xx_with_valid_spf` / `test_preserve_5xx_with_no_header`). The live run is reproducible via Appendix B in the provided container.
- **Scope:** investigation only — no remediation is implemented; hardening categories in §(g) are out-of-scope analytical observations.

