# SimpleLogin Email-Forwarding Pipeline — Runtime Investigation (Q1–Q4)

## Purpose

This document is a **read-only, runtime investigation** of the SimpleLogin inbound
email-forwarding pipeline, written to answer four specific questions about the
*actual generated values* observed when the real handler processes an inbound email
addressed to an alias. It was motivated by a production report of "inconsistent"
forwarding behavior. **Every value below was captured by running the real code path
(`email_handler.handle(envelope, msg)`), not inferred from reading it.**

To probe the reported "inconsistency" rigorously, the **byte-identical input** was
executed **twice, in two independent OS processes** (run 1, PID `407`, captured in
`run.log`; run 2, PID `431`, captured in `run2.log`) against the **same fixed alias**.
The *Run-to-run variability* section then separates the values that **vary despite
byte-identical input** (autoincrement ids, `created_at` timestamps, and the
`make_msgid` timestamp/pid/random at first mint) from the values that **only change
when the input itself changes** (e.g. the reverse-alias, which is generated once per
contact). This distinction is the crux of the answer to the reported inconsistency.

No SimpleLogin source, test, configuration, or migration file was modified. The only
artifact produced is this document.

---

## Environment & Invocation

All observations were produced against the **canonical build** for commit
`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (source branch `app_2cd6ee777f8c`), using
the **canonical `tests/test.env` configuration** (`EMAIL_DOMAIN=sl.local`,
`NOT_SEND_EMAIL=true`, `DB_URI=postgresql://test:test@localhost:15432/test`).

| Item | Value | Source of truth |
|------|-------|-----------------|
| Runtime | **Python 3.10** (venv reports `Python 3.10.18`) | `Dockerfile` `FROM python:3.10`; `pyproject.toml:61` `python = "^3.10"`; CI matrix `.github/workflows/main.yml` |
| Image | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (baked repo at `/app`, venv at `/app/venv`) | provided container |
| Config | `CONFIG=tests/test.env` | `tests/conftest.py:8-10` sets this before importing app modules |
| `EMAIL_DOMAIN` | `sl.local` | `tests/test.env:8`; consumed at `app/config.py:92` |
| `NOT_SEND_EMAIL` | `true` | `tests/test.env:7`; consumed at `app/config.py:91` |
| Database | **PostgreSQL** at **`localhost:15432`**, db/user/pw `test` — the **canonical `tests/test.env` DB URI, used with no override** | `tests/test.env:17` `DB_URI=postgresql://test:test@localhost:15432/test`; schema at Alembic head `32f25cbf12f6` (77 tables) |
| Cache / rate-limit | Redis (system service) `redis://localhost` | `tests/test.env:78` `MEM_STORE_URI` |

**Canonical DB note.** The provided image ships PostgreSQL as a system service; its
listen port was aligned to the canonical `tests/test.env` value **`15432`** (the
PostgreSQL **data directory — already at Alembic head `32f25cbf12f6`, 77 tables — was
left untouched**; only the listen port was set). The run therefore uses the
**unmodified `tests/test.env` `DB_URI` (`postgresql://test:test@localhost:15432/test`)
with no override**, matching the CI topology in `.github/workflows/main.yml`. This is
confirmed by the process's own startup line:

```text
[ENV] EMAIL_DOMAIN=sl.local NOT_SEND_EMAIL=True DB_URI=postgresql://test:test@localhost:15432/test
```

**Exact commands used** (run from the baked repo root `/app` inside the container):

```bash
# 1. bring up the backing services (system services in the canonical image)
service postgresql start && service redis-server start

# 2. schema is already at Alembic head in the image; the canonical migration command
#    is therefore a no-op (reports head 32f25cbf12f6 and applies nothing):
CONFIG=tests/test.env poetry run alembic upgrade head        # .github/workflows/main.yml

# 3. run the temporary observation script against the REAL handler, once PER PROCESS,
#    capturing stdout+stderr. No DB_URI override -> the canonical tests/test.env
#    DB_URI (port 15432) is used as-is.
cd /app
export PYTHONPATH=/app
export CONFIG=tests/test.env                                 # exactly as tests/conftest.py:8-10
export GITHUB_ACTIONS_TEST=true
export DKIM_PRIVATE_KEY_PATH=/tmp/sl_investigate/dkim.key    # see "DKIM key note" below
/app/venv/bin/python /tmp/sl_investigate/observe.py > /tmp/sl_investigate/run.log  2>&1   # run 1 (PID 407)
/app/venv/bin/python /tmp/sl_investigate/observe.py > /tmp/sl_investigate/run2.log 2>&1   # run 2 (PID 431)
```

Every quoted log block, header value, record and id below was produced by exactly this
`observe.py` invocation and taken verbatim from `run.log` (run 1) or `run2.log`
(run 2). The script set `CONFIG` itself before importing any app module
(`os.environ["CONFIG"] = "/app/tests/test.env"`), exactly as `tests/conftest.py:8-10`
does.

**Outbound mail was suppressed two ways**, so nothing left the process:
`NOT_SEND_EMAIL=true` (`app/config.py:91`) and the in-process capture hook
`mail_sender.store_emails_instead_of_sending()` (`app/mail_sender.py:102`), which
makes the real `send()` (`app/mail_sender.py:126`) retain the fully transformed
outgoing message in `mail_sender.get_stored_emails()` (`app/mail_sender.py:108`)
instead of transmitting it. The `From` (Q3) and `Message-ID` (Q2) values quoted below
are read from that retained `EmailMessage` — i.e. the exact bytes the code produced.

**DKIM key note (environment fix, no answer values affected).** The forward path
DKIM-signs the outgoing message (`email_handler.py:891` -> `app/email_utils.py`
`add_dkim_signature`). `dkimpy 1.0.5` requires a **PKCS#1** RSA key
(`-----BEGIN RSA PRIVATE KEY-----`), while the image's `local_data/dkim.key` is
**PKCS#8** (`-----BEGIN PRIVATE KEY-----`). The **same key material** was re-encoded to
PKCS#1 at a temp path and pointed to via `DKIM_PRIVATE_KEY_PATH`:

```bash
openssl rsa -in /app/local_data/dkim.key -traditional -out /tmp/sl_investigate/dkim.key
```

`local_data/dkim.key` was **not** modified. After this fix the canonical test
`tests/test_email_handler.py::test_preserve_headers` passes (`1 passed`) and the
forward completes to `E200`. The DKIM signature is not one of the four questions and
does not affect the `From` header, the `Message-ID`, the log text, or the database
records reported here.

### How the real path was exercised

The blessed way to drive the production handler without a live SMTP socket (used by
`tests/test_email_handler.py:82-85`) is to build an `aiosmtpd` `Envelope` + a standard
`EmailMessage` and call `email_handler.handle(envelope, msg)` (`email_handler.py:1945`).
Inside `handle()`, each recipient is routed (`email_handler.py:2195-2213`): a
reverse-alias recipient -> **reply** via `handle_reply` (`email_handler.py:966`), any
other recipient -> **forward** via `handle_forward` (`email_handler.py:536`). The
temporary script seeded one fixed `User` + default `Mailbox` (`tests/utils.py:17`
`create_new_user` pattern) and one **fixed-address** alias `probe-alias@sl.local`,
then exercised: **forward x2** (byte-identical input), **failure** (non-existent
alias), **reply x2** (mint then reuse), a **no-`Message-ID` reply x2** (fresh mint),
and a **different-sender** forward. The seeded objects observed in both runs:

```text
[SETUP] user.id=3 user.email=probeuser@mailbox.test include_sender_in_reverse_alias=True sender_format=0
[SETUP] alias.id=6 alias.email=probe-alias@sl.local mailbox.id=3 mailbox.email=probeuser@mailbox.test
```

The `SL` logger writes every line to stdout at `DEBUG` (logger name `SL`,
`app/log.py:79`; timestamps in **UTC**) using this exact format string
(`app/log.py:12-15`):

```python
_log_format = (
    "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
    '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
)
```

Because the format embeds `"%(pathname)s:%(lineno)d"`, **every captured log line below
is self-citing** — it carries the emitting `file:line` produced by the running code.
(The `%(message_id)s` slot is the Postfix queue id injected by `app/log.py:22`; it is
empty here because these messages were not delivered by Postfix, shown as the blank
` -  - ` between `funcName()` and the message.)

> A note on line numbers: the emitting line number is the line of the `LOG.d(...)`
> **call**. This document cites the **observed** call-site number that actually appears
> in the output.

---

## Q1 — Exact log text: successful forward vs. non-existent-alias failure

**Question:** *"what is the exact log message text that appears when an email is
successfully forwarded versus when it fails due to a non-existent alias."*

Both blocks below were produced by the run-1 command
`/app/venv/bin/python /tmp/sl_investigate/observe.py > /tmp/sl_investigate/run.log 2>&1`
(PID `407`) and are quoted unedited from `run.log`.

### Q1a — Successful forward (returns `E200`)

**Observed input line** (a plain message from a fixed external sender to the seeded
alias `probe-alias@sl.local`):

```text
[S1] OBSERVED INPUT LINE (fixed): from=probe.sender@external.example.com to=probe-alias@sl.local subject='Probe forward subject' message_id=<probe-original-message-id@external.example.com> body='Probe forward body.\n'
```

**Unedited captured `SL` log block for the successful forward** (from `run.log`):

```text
2026-07-08 22:20:07,486 - SL - DEBUG - 407 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 22:20:07,488 - SL - DEBUG - 407 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:probe.sender@external.example.com, rcpt_tos:['probe-alias@sl.local'], header_from:probe.sender@external.example.com, header_to:probe-alias@sl.local, cc:None, reply-to:None, message_id:<probe-original-message-id@external.example.com>, client_ip:None, headers:[('From', 'probe.sender@external.example.com'), ('To', 'probe-alias@sl.local'), ('Subject', 'Probe forward subject'), ('Message-ID', '<probe-original-message-id@external.example.com>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-08 22:20:07,492 - SL - DEBUG - 407 - "/app/email_handler.py:2202" - handle() -  - Forward phase probe.sender@external.example.com(probe.sender@external.example.com) -> probe-alias@sl.local
2026-07-08 22:20:07,499 - SL - DEBUG - 407 - "/app/email_handler.py:580" - handle_forward() -  - Create or get contact for from_header:probe.sender@external.example.com
2026-07-08 22:20:07,518 - SL - DEBUG - 407 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 4 probe.sender@external.example.com 6> for alias <Alias 6 probe-alias@sl.local> with email probe.sender@external.example.com invalid_email=False
2026-07-08 22:20:07,518 - SL - INFO - 407 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() -  - DMARC check disabled
2026-07-08 22:20:07,525 - SL - DEBUG - 407 - "/app/email_handler.py:688" - forward_email_to_mailbox() -  - Forward <Contact 4 probe.sender@external.example.com 6> -> <Alias 6 probe-alias@sl.local> -> <Mailbox 3 probeuser@mailbox.test>
2026-07-08 22:20:07,528 - SL - DEBUG - 407 - "/app/email_handler.py:740" - forward_email_to_mailbox() -  - Create <EmailLog 9> for <Contact 4 probe.sender@external.example.com 6>, <User 3 Probe User probeuser@mailbox.test>, <Mailbox 3 probeuser@mailbox.test>
2026-07-08 22:20:07,533 - SL - WARNING - 407 - "/app/email_handler.py:857" - forward_email_to_mailbox() -  - missing date header, create one
2026-07-08 22:20:07,534 - SL - DEBUG - 407 - "/app/email_handler.py:867" - forward_email_to_mailbox() -  - From header, new:"probe.sender at external.example.com" <probe_sender_at_external_example_com_ujtpemwyct@sl.local>, old:probe.sender@external.example.com
2026-07-08 22:20:07,534 - SL - DEBUG - 407 - "/app/email_handler.py:316" - replace_header_when_forward() -  - Delete Cc header, old value None
2026-07-08 22:20:07,535 - SL - DEBUG - 407 - "/app/email_handler.py:313" - replace_header_when_forward() -  - Replace To header, old: probe-alias@sl.local, new: probe-alias@sl.local
2026-07-08 22:20:07,535 - SL - INFO - 407 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() -  - Email has no unsubscribe header
2026-07-08 22:20:07,544 - SL - DEBUG - 407 - "/app/email_handler.py:893" - forward_email_to_mailbox() -  - Forward mail from probe.sender@external.example.com to probeuser@mailbox.test, mail_options:[], rcpt_options:[] 
2026-07-08 22:20:07,544 - SL - DEBUG - 407 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Probe forward subject', from '"probe.sender at external.example.com" <probe_sender_at_external_example_com_ujtpemwyct@sl.local>' to 'probe-alias@sl.local'
```

Return value of `handle()` (the SMTP status a real sender would receive):

```text
[S1][FWD1] handle() returned: '250 Message accepted for delivery'  stored_count=1
```

**The notable success lines and their emitting `file:line`:**

| Log message (verbatim) | Emitting `file:line` |
|------------------------|----------------------|
| `Forward phase %s(%s) -> %s` | `email_handler.py:2202` |
| `Create or get contact for from_header:%s` | `email_handler.py:580` |
| `Created contact <Contact …> for alias <Alias …> … invalid_email=%s` | `app/contact_utils.py:110` |
| `Forward %s -> %s -> %s` (contact -> alias -> mailbox) | `email_handler.py:688` |
| `Create %s for %s, %s, %s` (the `EmailLog`) | `email_handler.py:740` |
| `From header, new:%s, old:%s` | `email_handler.py:867` |
| `Forward mail from %s to %s, mail_options:%s, rcpt_options:%s ` | `email_handler.py:893` |
| `send email with subject '%s', from '%s' to '%s'` | `app/mail_sender.py:131` |
| returned `250 Message accepted for delivery` | `app/email/status.py:2` `E200` |

**Cause -> effect.** The recipient `probe-alias@sl.local` matches the seeded alias, so
`handle()` routes to `handle_forward` (`email_handler.py:536`), which creates/looks-up
the sender `Contact`, persists an `EmailLog`, rewrites the `From` header, DKIM-signs,
and calls `send()`. Because at least one delivery succeeds, the status aggregation at
`email_handler.py:2227-2231` returns that delivery's status — `status.E200`
(`"250 Message accepted for delivery"`, `app/email/status.py:2`).

### Q1b — Non-existent alias failure (returns `E515`)

To reliably reach the failure branch, the message was addressed to a syntactically
valid but non-existent address at `sl.local` with **no directory separator**, so
on-the-fly creation cannot rescue it.

**Observed input line:**

```text
[S3] OBSERVED INPUT LINE (fixed): from=probe.sender@external.example.com to=nonexistent-probe-fixed@sl.local subject='Probe failure subject' message_id=<probe-fail-message-id@external.example.com>
```

**Unedited captured `SL` log block for the failure** (from `run.log`):

```text
2026-07-08 22:20:08,005 - SL - DEBUG - 407 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 22:20:08,006 - SL - DEBUG - 407 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:probe.sender@external.example.com, rcpt_tos:['nonexistent-probe-fixed@sl.local'], header_from:probe.sender@external.example.com, header_to:nonexistent-probe-fixed@sl.local, cc:None, reply-to:None, message_id:<probe-fail-message-id@external.example.com>, client_ip:None, headers:[('From', 'probe.sender@external.example.com'), ('To', 'nonexistent-probe-fixed@sl.local'), ('Subject', 'Probe failure subject'), ('Message-ID', '<probe-fail-message-id@external.example.com>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-08 22:20:08,010 - SL - DEBUG - 407 - "/app/email_handler.py:2202" - handle() -  - Forward phase probe.sender@external.example.com(probe.sender@external.example.com) -> nonexistent-probe-fixed@sl.local
2026-07-08 22:20:08,017 - SL - DEBUG - 407 - "/app/email_handler.py:545" - handle_forward() -  - alias nonexistent-probe-fixed@sl.local not exist. Try to see if it can be created on the fly
2026-07-08 22:20:08,022 - SL - INFO - 407 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() -  - Cannot auto-create custom domain alias for nonexistent-probe-fixed@sl.local because there's no custom domain for sl.local
2026-07-08 22:20:08,022 - SL - INFO - 407 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() -  - Cannot auto-create nonexistent-probe-fixed@sl.local since it has no directory separator
2026-07-08 22:20:08,022 - SL - DEBUG - 407 - "/app/email_handler.py:551" - handle_forward() -  - alias nonexistent-probe-fixed@sl.local cannot be created on-the-fly, return 550
```

Return value of `handle()`:

```text
[S3][FAIL] handle() returned: '550 SL E515 Email not exist'  stored_count=0
```

**The notable failure lines and their emitting `file:line`:**

| Log message (verbatim) | Emitting `file:line` |
|------------------------|----------------------|
| `alias %s not exist. Try to see if it can be created on the fly` | `email_handler.py:545` |
| `Cannot auto-create custom domain alias for … because there's no custom domain for sl.local` | `app/alias_utils.py:104` |
| `Cannot auto-create … since it has no directory separator` | `app/alias_utils.py:165` |
| `alias %s cannot be created on-the-fly, return 550` | `email_handler.py:551` |
| returned `550 SL E515 Email not exist` | `app/email/status.py:51` `E515` |

**Cause -> effect.** No alias matches `nonexistent-probe-fixed@sl.local`, so
`handle_forward` logs "not exist" (`email_handler.py:545`) and calls `try_auto_create`.
Auto-creation requires either a catch-all custom domain or a directory address
(`app/alias_utils.py`); a freshly seeded user has neither, so both checks decline (the
two `alias_utils` `INFO` lines) and `try_auto_create` returns `None`. `handle_forward`
then logs the "cannot be created on-the-fly" line (`email_handler.py:551`) and returns
`status.E515` (`email_handler.py:555`). Every delivery failed, so the aggregation
returns the first failure — `status.E515` (`"550 SL E515 Email not exist"`,
`app/email/status.py:51`). No message was stored (`stored_count=0`), confirming nothing
was forwarded.

**The exact difference (success vs. failure).** A successful forward emits the
`Forward %s -> %s -> %s`, `Create %s …` (`EmailLog`), `From header, new:… old:…`, and
`Forward mail from … to …` `DEBUG` lines and returns `250 Message accepted for
delivery`. A non-existent-alias failure emits `alias … not exist …`, the two
`alias_utils` auto-create refusals, and `alias … cannot be created on-the-fly, return
550`, and returns `550 SL E515 Email not exist`.

---

## Q2 — The SL `Message-ID` vs. the original `Message-ID`

**Question:** *"what specific SL Message-ID gets generated during forwarding and how
does it differ from the original Message-ID."*

**Key finding (the honest answer):** during a **pure forward, no SL Message-ID is
generated at all** — the sender's original `Message-ID` is *preserved* on the outgoing
message and stored in `EmailLog.message_id`, while `EmailLog.sl_message_id` stays
`NULL`. A new **SL Message-ID is minted only in the reply phase**, by
`make_msgid(str(email_log.id), get_email_domain_part(alias.email))`
(`email_handler.py:1311-1313`). Both paths were exercised.

### Q2a — Forward preserves the original `Message-ID`

The message was sent with `Message-ID: <probe-original-message-id@external.example.com>`.
The captured outgoing (forwarded) message carries the **same** value, unchanged, and
the persisted `EmailLog` stores the original with an empty SL column:

```text
[S1][FWD1] OUT Message-ID= '<probe-original-message-id@external.example.com>'
[S1][FWD1][ORM] EmailLog id=9 created_at=<Arrow [2026-07-08T22:20:07.526751+00:00]> is_reply=False contact_id=4 mailbox_id=3 message_id='<probe-original-message-id@external.example.com>' sl_message_id=None MessageIDMatching=[]
```

*Why:* `forward_email_to_mailbox` persists
`EmailLog.create(..., message_id=str(msg[headers.MESSAGE_ID]), ...)`
(`email_handler.py:737`); the forward keeps the `Message-ID` header. Hence the
forwarded `Message-ID` equals the sender's original and `sl_message_id` is `NULL`.

### Q2b — Reply mints the SL `Message-ID`

When the mailbox replies to the reverse-alias, `handle_reply` calls
`replace_original_message_id` (`email_handler.py:1296`), which mints the SL id via
`make_msgid` and logs it (`email_handler.py:1314`):

```text
2026-07-08 22:20:08,171 - SL - DEBUG - 407 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178354920817.407.12528124575623281356.11@sl.local>
```

The captured outgoing (relayed) reply message carries that minted id, and a
`MessageIDMatching` row records the mapping (`email_handler.py:1316`):

```text
[S2][REPLY1] OUT Message-ID (minted SL id) = '<178354920817.407.12528124575623281356.11@sl.local>'
[S2][REPLY1][ORM] MessageIDMatching id=2 created_at=<Arrow [2026-07-08T22:20:08.172219+00:00]> sl_message_id='<178354920817.407.12528124575623281356.11@sl.local>' original_message_id='<probe-reply-message-id@mailbox.test>' email_log_id=11
```

**Reply with the SAME reply `Message-ID` REUSES the SL id (it is NOT re-minted).**
`replace_original_message_id` first looks up `MessageIDMatching` by the original
`Message-ID`; if found it reuses the stored `sl_message_id` and logs
(`email_handler.py:1309`):

```text
2026-07-08 22:20:08,470 - SL - DEBUG - 407 - "/app/email_handler.py:1309" - replace_original_message_id() -  - reuse the sl_message_id <178354920817.407.12528124575623281356.11@sl.local>
[S2][REPLY2] OUT Message-ID = '<178354920817.407.12528124575623281356.11@sl.local>'
[S2][CMP] REPLY1 SL id == REPLY2 SL id ? True  (reused for identical original_message_id)
```

So the SL Message-ID for a given reply `Message-ID` is minted **once** and then
**stored and reused** — it is *not* nondeterministic per identical input. (The
*Run-to-run variability* section shows run 2, a separate process, reusing this very
id byte-for-byte.)

**When the SL id genuinely varies per identical input: `make_msgid` at first mint.**
A reply carrying **no** `Message-ID` header hits the `else` branch
(`email_handler.py:1333-1336`), which mints a fresh id **every** call (no dedup).
Sending the byte-identical no-`Message-ID` reply twice produced two **different** SL
ids:

```text
[S2b] no-Message-ID reply #1 minted SL id = '<178354920863.407.10570044442653433679.13@sl.local>'
[S2b] no-Message-ID reply #2 minted SL id = '<178354920866.407.9639224699420665195.14@sl.local>'
[S2b][CMP] identical input, SL ids differ ? True (make_msgid is nondeterministic)
```

### The structural difference

`make_msgid(idstring, domain)` (Python `email.utils`, imported at `email_handler.py:42`)
produces `<{int(time.time()*100)}.{pid}.{rand}.{idstring}@{domain}>`. Here
`idstring = str(email_log.id)` and `domain = get_email_domain_part(alias.email)`
(`app/email_utils.py:448`) = the **alias/SL domain** `sl.local`. Decomposing the
observed minted SL id `<178354920817.407.12528124575623281356.11@sl.local>`:

| Component | Observed value | Meaning |
|-----------|----------------|---------|
| timestamp | `178354920817` | `int(time.time()*100)` at mint time |
| pid | `407` | the emitting process id (matches the `- 407 -` field of that run) |
| random | `12528124575623281356` | random component from `make_msgid` |
| **idstring** | `11` | **`str(email_log.id)`** — the reply's `EmailLog.id` (11) |
| domain | `sl.local` | the alias/SL domain, not the sender's |

Contrast with the original `Message-ID` (`<probe-original-message-id@external.example.com>`):
a free-form local-part at the **sender's** domain, chosen entirely by the sending MTA.
The two differ in **who owns the domain** (sender's domain vs. `sl.local`), in
**structure** (free-form vs. the fixed four-token `time.pid.rand.emaillog_id` shape),
and in the fact that the SL id **embeds the SimpleLogin `EmailLog.id`** as its last
token (observed `EmailLog id=11` -> SL id ends `.11@sl.local`).

---

## Q3 — The transformed `From` header (including the reverse-alias `reply_email` format)

**Question:** *"what is the exact From header value in the forwarded email after
transformation including the reply-email address format."*

The forward replaces the sender's `From` with `contact.new_addr()`
(`email_handler.py:864-867`; `Contact.new_addr` at `app/models.py:2008`). **The exact,
unedited transformed `From` value captured on the outgoing forwarded message** (run 1):

```text
[S1][FWD1] OUT From      = '"probe.sender at external.example.com" <probe_sender_at_external_example_com_ujtpemwyct@sl.local>'
```

The same value appears in the emitting log line (`email_handler.py:867`), part of the Q1a block above:

```text
2026-07-08 22:20:07,534 - SL - DEBUG - 407 - "/app/email_handler.py:867" - forward_email_to_mailbox() -  - From header, new:"probe.sender at external.example.com" <probe_sender_at_external_example_com_ujtpemwyct@sl.local>, old:probe.sender@external.example.com
```

### Anatomy of the value

The `From` is a display-name + reverse-alias address, RFC-2047 encoded via
`sl_formataddr` (`app/models.py:2044-2046`):

- **Display name — `"probe.sender at external.example.com"`.** This is the default
  `sender_format = AT` rendering. `Contact.new_addr` reads
  `sender_format = user.sender_format if user else SenderFormatEnum.AT.value`
  (`app/models.py:2019`); the seeded user has `sender_format=0` (observed
  `[SETUP] … sender_format=0`), and `SenderFormatEnum.AT = 0`. The AT branch
  (`app/models.py:2028-2034`) builds the display name by replacing `@` with `" at "`
  in the contact's `website_email`; because this contact has no distinct display name
  (`name=None`, see Q4), the name is exactly the formatted email
  `probe.sender at external.example.com`.
- **Address — `probe_sender_at_external_example_com_ujtpemwyct@sl.local`.** This is the
  contact's `reply_email` (the reverse-alias), from `generate_reply_email(contact_email,
  alias)` (`app/email_utils.py:1103`).

### Observed `reply_email` format and why it takes the sender-inclusive branch

`generate_reply_email` has two branches. The observed value takes the
**sender-inclusive** branch (`app/email_utils.py:1137-1143`), **not** the "20–50 random
characters" default:

```python
reply_email = f"{contact_email}_{random_string(random_length)}@{reply_domain}"   # app/email_utils.py:1142, random_length = randint(5, 10)
```

*Why (verified at runtime, not assumed):* the branch is gated by
`user.include_sender_in_reverse_alias` (`app/email_utils.py:1116-1117`). This column has
an ORM-side `default=True` (`app/models.py:455-457`). The canonical seeding helper
creates the user through the ORM, so the default `True` applies — confirmed by the
seeded user in **both** runs:

```text
[SETUP] user.id=3 user.email=probeuser@mailbox.test include_sender_in_reverse_alias=True sender_format=0    <- include_sender_in_reverse_alias=True
```

Consequently `contact_email` is sanitized (`app/email_utils.py:1120-1126`: `@`->`_at_`,
`.`->`_`, alphanumeric) and used as a prefix. For
`probe.sender@external.example.com` this yields `probe_sender_at_external_example_com`,
followed by `_` + a random token of length `randint(5, 10)`
(`random_string`, `app/utils.py:41` = lowercase letters), at
`reply_domain = config.EMAIL_DOMAIN = sl.local`.

> If a user instead had `include_sender_in_reverse_alias = False`, the code takes the
> default branch (`app/email_utils.py:1143-1148`)
> `reply_email = f"{random_string(random_length)}@{reply_domain}"` with
> `random_length = randint(20, 50)` — i.e. `<20–50 random lowercase chars>@sl.local`,
> with **no** sender prefix. That branch was **not** exercised here because the
> canonical seeded user has the flag `True`.

**The reverse-alias is stable per contact, and only differs for a different sender.**
The exact same fixed input produced the byte-identical `From`/reverse-alias in run 1
and run 2 (see *Run-to-run variability*). It differs **only when the sender changes**,
because a different sender is a **different `Contact`**, and the random token is drawn
once when that contact is created. Observed with a deliberately changed sender:

```text
[S4] OBSERVED INPUT LINE (sender CHANGED): from=probe.sender2@external.example.com to=probe-alias@sl.local subject='Probe forward subject' message_id=<probe-original-message-id@external.example.com>
[S4] OUT From = '"probe.sender2 at external.example.com" <probe_sender2_at_external_example_com_dvqtnbujo@sl.local>'
[S4] reverse-alias(sender1) = 'probe_sender_at_external_example_com_ujtpemwyct@sl.local'
[S4] reverse-alias(sender2) = 'probe_sender2_at_external_example_com_dvqtnbujo@sl.local'
[S4][CMP] reverse-alias differs because SENDER (input) changed -> new contact: True
```

**Cause -> effect (and the SimpleLogin model).** The forward masks the real sender: the
outgoing `From` is a **reverse-alias** address at `sl.local` (never the sender's real
address), decorated with a human-readable "who this is from" display name in the `AT`
format. This matches SimpleLogin's documented reverse-alias behavior — mail arrives
*from* a reverse-alias, and replying to it relays back to the original sender with the
user's real address masked — but the concrete address above is the value produced by
this run.

---

## Q4 — Database records created by a single forward (real IDs + timestamps)

**Question:** *"what database records are created during a single forward operation
show me the actual record IDs and timestamps."*

A single forward **from a new sender** persists **three** rows — a `Contact`, a
`user_audit_log` (the audit trail of the contact creation), and an `EmailLog`. This
was established **empirically**, not by reading the code: the observation script
snapshotted the row count of **all 77 public tables** immediately before and after a
single `handle()` forward and printed exactly which tables gained rows. The unedited
diff (run 1; sender `probe.sender@external.example.com` -> alias `probe-alias@sl.local`):

```text
    table 'contact': +1 row(s); new rows (id > 1):
      RAW contact: {'id': 4, 'created_at': datetime.datetime(2026, 7, 8, 22, 20, 7, 510009), 'updated_at': None, 'alias_id': 6, 'website_email': 'probe.sender@external.example.com', 'reply_email': 'probe_sender_at_external_example_com_ujtpemwyct@sl.local', 'website_from': None, 'user_id': 3, 'is_cc': False, 'name': None, 'pgp_finger_print': None, 'pgp_public_key': None, 'mail_from': 'probe.sender@external.example.com', 'invalid_email': False, 'block_forward': False, 'automatic_created': True, 'flags': 0}
    table 'email_log': +1 row(s); new rows (id > 1):
      RAW email_log: {'id': 9, 'created_at': datetime.datetime(2026, 7, 8, 22, 20, 7, 526751), 'updated_at': None, 'contact_id': 4, 'is_reply': False, 'blocked': False, 'bounced': False, 'refused_email_id': None, 'user_id': 3, 'is_spam': False, 'spam_status': None, 'bounced_mailbox_id': None, 'spam_score': None, 'mailbox_id': 3, 'spam_report': None, 'auto_replied': False, 'alias_id': 6, 'message_id': '<probe-original-message-id@external.example.com>', 'sl_message_id': None}
    table 'user_audit_log': +1 row(s); new rows (id > 1):
      RAW user_audit_log: {'id': 4, 'created_at': datetime.datetime(2026, 7, 8, 22, 20, 7, 515405), 'updated_at': None, 'user_id': 3, 'user_email': 'probeuser@mailbox.test', 'action': 'create_contact', 'message': 'Created contact 4 (probe.sender@external.example.com)'}
```

Exactly **three** tables gained one row each — `contact`, `user_audit_log`, and
`email_log` — and **no other table changed** (in particular, **no `message_id_matching`
row**). The same three rows, read back through the ORM with typed `id` / Arrow
`created_at`:

```text
[S1][FWD1][ORM] Contact id=4 created_at=<Arrow [2026-07-08T22:20:07.510009+00:00]> user_id=3 alias_id=6 website_email='probe.sender@external.example.com' reply_email='probe_sender_at_external_example_com_ujtpemwyct@sl.local' name=None
[S1][FWD1][ORM] UserAuditLog id=4 created_at=<Arrow [2026-07-08T22:20:07.515405+00:00]> user_id=3 user_email='probeuser@mailbox.test' action='create_contact' message='Created contact 4 (probe.sender@external.example.com)'
[S1][FWD1][ORM] EmailLog id=9 created_at=<Arrow [2026-07-08T22:20:07.526751+00:00]> is_reply=False contact_id=4 mailbox_id=3 message_id='<probe-original-message-id@external.example.com>' sl_message_id=None MessageIDMatching=[]
```

| Table | `id` | `created_at` (UTC) | Key columns |
|-------|------|--------------------|-------------|
| `contact` (`app/models.py:1863`) | **4** | **2026-07-08T22:20:07.510009+00:00** | `website_email=probe.sender@external.example.com`, `reply_email=probe_sender_at_external_example_com_ujtpemwyct@sl.local`, `name=None`, `automatic_created=True` |
| `user_audit_log` (`app/models.py:3829`) | **4** | **2026-07-08T22:20:07.515405+00:00** | `user_id=3`, `user_email=probeuser@mailbox.test`, `action=create_contact`, `message=Created contact 4 (probe.sender@external.example.com)` |
| `email_log` (`app/models.py:2060`) | **9** | **2026-07-08T22:20:07.526751+00:00** | `is_reply=False`, `message_id=<probe-original-message-id@external.example.com>`, `sl_message_id=None` |
| `message_id_matching` (`app/models.py:3365`) | — | — | **none created on the forward path** |

**The `user_audit_log` row is created inside `create_contact`.** After
`Contact.create(...)` (`app/contact_utils.py:92`), `create_contact` calls
`emit_user_audit_log(user=alias.user, action=UserAuditLogAction.CreateContact,
message=f"Created contact {contact.id} ({contact.email})", commit=True)`
(`app/contact_utils.py:104-109`), which persists a `UserAuditLog`
(`app/user_audit_log_utils.py:35-44` -> `UserAuditLog.create(...)`;
model `app/models.py:3829-3843`, `action` value `create_contact` per
`UserAuditLogAction.CreateContact`, `app/user_audit_log_utils.py:23`). This row is a
genuine database side effect of the forward and is therefore part of the answer.

**Order of creation (timestamps).** `Contact` (`.510009`) -> `user_audit_log`
(`.515405`) -> `EmailLog` (`.526751`), because `get_or_create_contact`
(`email_handler.py:180` -> `app/contact_utils.py:42`) — which creates the contact and
then emits the audit row — runs before `EmailLog.create(...)` (`email_handler.py:732`).

**A repeat forward from the *same* sender persists only ONE row.** The `Contact` and
its `user_audit_log` are created once (at first contact); a subsequent byte-identical
forward reuses them and adds only a new `EmailLog`. The all-tables diff around the
immediately-following identical forward (FWD#2) confirms this — only `email_log`
changed:

```text
    table 'email_log': +1 row(s); new rows (id > 9):
      RAW email_log: {'id': 10, 'created_at': datetime.datetime(2026, 7, 8, 22, 20, 7, 854361), 'updated_at': None, 'contact_id': 4, 'is_reply': False, 'blocked': False, 'bounced': False, 'refused_email_id': None, 'user_id': 3, 'is_spam': False, 'spam_status': None, 'bounced_mailbox_id': None, 'spam_score': None, 'mailbox_id': 3, 'spam_report': None, 'auto_replied': False, 'alias_id': 6, 'message_id': '<probe-original-message-id@external.example.com>', 'sl_message_id': None}
[S1][FWD2][ORM] EmailLog id=10 created_at=<Arrow [2026-07-08T22:20:07.854361+00:00]> is_reply=False contact_id=4 mailbox_id=3 message_id='<probe-original-message-id@external.example.com>' sl_message_id=None MessageIDMatching=[]
```

**Shape of `id` / `created_at`.** All three tables inherit `ModelMixin`
(`app/models.py:62-65`): `id` is an autoincrement integer PK, `created_at` is an
`ArrowType` defaulting to `arrow.utcnow` (hence the `+00:00`/UTC offset), and
`updated_at`. The ids are small (`4`, `9`, …) because the run started from a clean,
freshly-seeded baseline; they are the genuine autoincrement values observed.

**No `MessageIDMatching` on forward; one per new reply `Message-ID`.** The forward
diffs above show no `message_id_matching` row. A `MessageIDMatching` row
(`app/models.py:3365`) appears **only** on the reply path, created at
`email_handler.py:1316`. The final dump of all `EmailLog` rows for the alias (run 1)
shows the two forward logs (`is_reply=False`, `sl_message_id=None`, empty
`MessageIDMatching`) vs. the reply logs — the first reply mints and owns the matching
row (`id=2`), the second reply (same reply `Message-ID`) reuses the SL id and has an
empty list, and the two no-`Message-ID` replies each mint a fresh SL id with no matching
row:

```text
[FINAL] EmailLog id=9 created_at=<Arrow [2026-07-08T22:20:07.526751+00:00]> is_reply=False contact_id=4 mailbox_id=3 message_id='<probe-original-message-id@external.example.com>' sl_message_id=None MessageIDMatching=[]
[FINAL] EmailLog id=10 created_at=<Arrow [2026-07-08T22:20:07.854361+00:00]> is_reply=False contact_id=4 mailbox_id=3 message_id='<probe-original-message-id@external.example.com>' sl_message_id=None MessageIDMatching=[]
[FINAL] EmailLog id=11 created_at=<Arrow [2026-07-08T22:20:08.160682+00:00]> is_reply=True contact_id=4 mailbox_id=3 message_id='<probe-reply-message-id@mailbox.test>' sl_message_id='<178354920817.407.12528124575623281356.11@sl.local>' MessageIDMatching=[(2, '<178354920817.407.12528124575623281356.11@sl.local>', '<probe-reply-message-id@mailbox.test>')]
[FINAL] EmailLog id=12 created_at=<Arrow [2026-07-08T22:20:08.460987+00:00]> is_reply=True contact_id=4 mailbox_id=3 message_id='<probe-reply-message-id@mailbox.test>' sl_message_id='<178354920817.407.12528124575623281356.11@sl.local>' MessageIDMatching=[]
[FINAL] EmailLog id=13 created_at=<Arrow [2026-07-08T22:20:08.622671+00:00]> is_reply=True contact_id=4 mailbox_id=3 message_id=None sl_message_id='<178354920863.407.10570044442653433679.13@sl.local>' MessageIDMatching=[]
[FINAL] EmailLog id=14 created_at=<Arrow [2026-07-08T22:20:08.655550+00:00]> is_reply=True contact_id=4 mailbox_id=3 message_id=None sl_message_id='<178354920866.407.9639224699420665195.14@sl.local>' MessageIDMatching=[]
[FINAL] EmailLog id=15 created_at=<Arrow [2026-07-08T22:20:08.721359+00:00]> is_reply=False contact_id=5 mailbox_id=3 message_id='<probe-original-message-id@external.example.com>' sl_message_id=None MessageIDMatching=[]
```

---

## Run-to-run variability (this is the reported "inconsistency")

**Methodology.** The **byte-identical message input** (fixed sender, fixed alias, fixed
`Message-ID`, fixed subject and body) was executed in **two independent processes**
(run 1 PID `407` -> `run.log`; run 2 PID `431` -> `run2.log`). Comparing them isolates
values that **vary despite unchanged input** from values that are **stable** (and thus
only ever change when the input itself changes). Both runs were produced by the same
command shown in *Environment & Invocation*, differing only in the output file.

**Dedicated run-2 output block** (unedited, from `run2.log` — the same fixed input as
run 1):

```text
################ OBSERVE START pid=431 CONFIG=/app/tests/test.env ################
[ENV] EMAIL_DOMAIN=sl.local NOT_SEND_EMAIL=True DB_URI=postgresql://test:test@localhost:15432/test
[SETUP] user.id=3 user.email=probeuser@mailbox.test include_sender_in_reverse_alias=True sender_format=0
[SETUP] alias.id=6 alias.email=probe-alias@sl.local mailbox.id=3 mailbox.email=probeuser@mailbox.test

# --- Q1 forward (identical byte-for-byte input as run 1) ---
[S1] OBSERVED INPUT LINE (fixed): from=probe.sender@external.example.com to=probe-alias@sl.local subject='Probe forward subject' message_id=<probe-original-message-id@external.example.com> body='Probe forward body.\n'
[S1][FWD1] OUT From      = '"probe.sender at external.example.com" <probe_sender_at_external_example_com_ujtpemwyct@sl.local>'
[S1][FWD1] OUT Message-ID= '<probe-original-message-id@external.example.com>'
[S1] reverse-alias (contact.reply_email) = 'probe_sender_at_external_example_com_ujtpemwyct@sl.local'
[S1][CMP] From identical? True | Message-ID identical? True | reverse-alias identical? True
[S1][CMP] EmailLog id FWD1=16 FWD2=17 (differ=True) ; created_at FWD1=<Arrow [2026-07-08T22:20:19.562864+00:00]> FWD2=<Arrow [2026-07-08T22:20:19.893461+00:00]>

# --- Q2 reply with the SAME reply Message-ID -> SL id REUSED across processes ---
# (note the reused SL id still carries run 1's pid 407 and EmailLog id 11)
2026-07-08 22:20:20,209 - SL - DEBUG - 431 - "/app/email_handler.py:1309" - replace_original_message_id() -  - reuse the sl_message_id <178354920817.407.12528124575623281356.11@sl.local>
[S2][REPLY1] OUT Message-ID (minted SL id) = '<178354920817.407.12528124575623281356.11@sl.local>'

# --- make_msgid variance: no-Message-ID replies mint fresh (run 2 pid 431) ---
[S2b] no-Message-ID reply #1 minted SL id = '<178354922066.431.9891528518334241628.20@sl.local>'
[S2b] no-Message-ID reply #2 minted SL id = '<178354922069.431.5970494084105320899.21@sl.local>'
```

The run-2 block makes the key point concretely: the outgoing **`From`/reverse-alias**
and the **preserved forward `Message-ID`** are **byte-identical to run 1**, and the
reply's **SL Message-ID is reused verbatim** — the reused id
`<178354920817.407.12528124575623281356.11@sl.local>` still carries **run 1's** pid
(`407`) and `EmailLog.id` (`11`) even though run 2 is a different process (pid `431`).
Only the per-row autoincrement ids/timestamps and the fresh no-`Message-ID` mints
(now pid `431`) differ.

| Aspect | Run 1 (PID 407) | Run 2 (PID 431) | Category |
|--------|-----------------|-----------------|----------|
| Forward success status | `250 Message accepted for delivery` | `250 Message accepted for delivery` | **B — stable** (`E200`) |
| Failure status | `550 SL E515 Email not exist` | `550 SL E515 Email not exist` | **B — stable** (`E515`) |
| Emitting `file:line` of key log lines | `:2202`,`:545`,`:551`,`:867`,`:1314`,`:1309` | identical | **B — stable** |
| Log message templates | `Forward phase …`, `From header, new:… old:…`, `create a new sl_message_id …` | identical | **B — stable** |
| Forwarded `Message-ID` (preserved) | `<probe-original-message-id@external.example.com>` | `<probe-original-message-id@external.example.com>` | **B — stable** (= the input) |
| Transformed `From` / reverse-alias | `"…" <probe_sender_at_external_example_com_ujtpemwyct@sl.local>` | **identical** | **B — stable** (per-contact) |
| Reply SL `Message-ID` (same reply `Message-ID`) | `<178354920817.407.12528124575623281356.11@sl.local>` | **identical (reused)** | **B — stable** (stored & reused) |
| Autoincrement ids (fwd EmailLog #1 / #2) | `9` / `10` | `16` / `17` | **A — varies** (autoincrement) |
| `created_at` timestamps | `…T22:20:07.*` | `…T22:20:19.*` | **A — varies** (`arrow.utcnow`) |
| Process `pid` | `407` | `431` | **A — varies** (process id) |
| no-`Message-ID` reply SL id #1 / #2 | `…407…13`, `…407…14` | `…431…20`, `…431…21` | **A — varies** (`make_msgid` fresh mint) |
| reverse-alias for a **different** sender | `…_ujtpemwyct@…` (sender 1) vs `…_dvqtnbujo@…` (sender 2) | same pattern | **C — changes only because the input (sender) changed** |

**Summary of the "inconsistency."** With **truly unchanged input**, the observable
header values are **stable**: the forward always returns `E200`, the non-existent alias
always returns `E515`, the forward always preserves the original `Message-ID`, the
`From` is always the same reverse-alias in the `AT` format, and a reply with the same
`Message-ID` always yields the same (reused) SL id. The values that genuinely **vary
despite identical input** are the **autoincrement ids**, the **`created_at`
timestamps**, and the **`make_msgid` timestamp/pid/random at first mint** (observable
directly only when an id is minted fresh, e.g. a no-`Message-ID` reply). The
reverse-alias and per-reply SL id that a production operator perceives as "different
every time" change because the **input differs** (a different sender is a different
`Contact` with its own one-time random reverse-alias; a different reply `Message-ID`
triggers a fresh mint) — **not** because the pipeline is nondeterministic for the same
input. Any monitor that pins these exact strings will perceive "inconsistency";
comparisons should instead assert the stable structure (status codes, header shapes,
`sl.local` domain, and the `EmailLog.id`-embedding rule).

---

## Cleanup & integrity

- The observation ran inside a disposable container started from the canonical image.
  The temporary script (`/tmp/sl_investigate/observe.py`), its logs
  (`/tmp/sl_investigate/run.log`, `run2.log`), and the temporary PKCS#1 DKIM key
  (`/tmp/sl_investigate/dkim.key`) live **outside** the repository and were removed
  after capture; the container and its PostgreSQL/Redis were torn down.
- **No SimpleLogin source, test, configuration, or migration file was modified**; in
  particular `local_data/dkim.key` was left untouched (the PKCS#1 re-encoding was
  written to a temp path and selected via `DKIM_PRIVATE_KEY_PATH` only), and the
  PostgreSQL **data directory** (schema at Alembic head `32f25cbf12f6`) was never
  dropped or migrated — only the listen port was aligned to the canonical `15432`.
- The only change to the repository is this new documentation file,
  `blitzy/documentation/app_2cd6ee777f8c.md`. `git status` reports a clean working tree
  apart from the new `blitzy/` directory.

### Appendix — `file:line` references cited (verified at commit `2cd6ee777f8c…`)

- Entry point / routing / aggregation: `email_handler.py:1945` (`handle`), `:2195-2213`
  (Reply/Forward dispatch), `:2202` (Forward phase log), `:2227-2231` (status aggregation).
- Forward: `email_handler.py:536` (`handle_forward`), `:545`/`:551`/`:555` (not-exist ->
  `E515`), `:580` (create/get contact), `:679` (`forward_email_to_mailbox`), `:688`,
  `:732`/`:737` (`EmailLog.create`, `message_id`), `:740`, `:864-867` (`From` rewrite),
  `:893`; auto-create refusals `app/alias_utils.py:104`/`:165`.
- Reply / SL Message-ID: `email_handler.py:966` (`handle_reply`), `:1296`
  (`replace_original_message_id`), `:1309` (reuse), `:1311-1314` (`make_msgid` + "create
  a new sl_message_id"), `:1316` (`MessageIDMatching.create`), `:1333-1336` (no-original
  mint); `:42` (`make_msgid` import); `app/email_utils.py:448` (`get_email_domain_part`).
- Models: `app/models.py:62-65` (`ModelMixin` id/created_at/updated_at), `:1863`
  (`Contact`), `:2008` (`Contact.new_addr`, AT branch `:2028-2034`, `sl_formataddr`
  `:2044-2046`), `:2060` (`EmailLog`), `:3365` (`MessageIDMatching`; `sl_message_id`
  unique `:3371`, `original_message_id` unique `:3372`), `:3829-3843` (`UserAuditLog`).
- Contact creation + audit log: `email_handler.py:180` (`get_or_create_contact`),
  `app/contact_utils.py:42` (`create_contact`), `:89`/`:92` (`generate_reply_email`,
  `Contact.create`), `:104-109` (`emit_user_audit_log`), `:110` (LOG.d);
  `app/user_audit_log_utils.py:23` (`CreateContact="create_contact"`), `:35-44`
  (`emit_user_audit_log` -> `UserAuditLog.create`).
- Reverse-alias: `app/email_utils.py:1103` (`generate_reply_email`), `:1116-1117`
  (flag read), `:1120-1126` (sanitize), `:1129` (`reply_domain`), `:1137-1143`
  (sender-inclusive, `:1142`), `:1143-1148` (default); `app/utils.py:41`
  (`random_string`); `app/models.py:455-457` (`include_sender_in_reverse_alias` default
  `True`), `:2019` (`sender_format`), `SenderFormatEnum.AT=0`.
- Logging / status / capture: `app/log.py:12-15` (format), `:22` (`set_message_id`),
  `:79` (logger `SL`); `app/email/status.py:2` (`E200`), `:51` (`E515`);
  `app/mail_sender.py:102` (`store_emails_instead_of_sending`), `:108`
  (`get_stored_emails`), `:126`/`:131` (`send`).
- Seeding / config: `tests/utils.py:17` (`create_new_user`); `tests/conftest.py:8-10`
  (`CONFIG`); `app/config.py:91`/`:92` (`NOT_SEND_EMAIL`/`EMAIL_DOMAIN`), `:192`
  (`DB_URI`); `tests/test.env:7`/`:8`/`:17`; `.github/workflows/main.yml` (CI: Postgres
  @15432, `alembic upgrade head`).
