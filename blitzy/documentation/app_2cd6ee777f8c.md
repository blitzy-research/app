# SimpleLogin Email-Forwarding Pipeline — Runtime Investigation (Q1–Q4)

## Purpose

This document is a **read-only, runtime investigation** of the SimpleLogin inbound
email‑forwarding pipeline, written to answer four specific questions about the
*actual generated values* observed when the real handler processes an inbound email
addressed to an alias. It was motivated by a production report of "inconsistent"
forwarding behavior. **Every value below was captured by running the real code path
(`email_handler.handle(envelope, msg)`), not inferred from reading it.** Where a
value legitimately varies from run to run, the same input was executed **twice in
two independent processes** and the stable structure is separated from the varying
components (see *Run‑to‑run variability*, which explains the reported "inconsistency").

No SimpleLogin source, test, configuration, or migration file was modified. The only
artifact produced is this document.

---

## Environment & Invocation

All observations were produced against the **canonical build** for commit
`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (source branch `app_2cd6ee777f8c`).

| Item | Value | Source of truth |
|------|-------|-----------------|
| Runtime | **Python 3.10** (venv reported `Python 3.10.18`) | `Dockerfile:8` `FROM python:3.10`; `pyproject.toml:61` `python = "^3.10"`; CI matrix `.github/workflows/main.yml` |
| Image | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (baked repo at `/app`) | provided container |
| Config | `CONFIG=tests/test.env` | `tests/conftest.py:8-10` sets this before importing app modules |
| `EMAIL_DOMAIN` | `sl.local` | `tests/test.env:8`; consumed at `app/config.py:92` `EMAIL_DOMAIN = os.environ["EMAIL_DOMAIN"].lower()` |
| `NOT_SEND_EMAIL` | `true` | `tests/test.env:7`; consumed at `app/config.py:91` |
| Database | PostgreSQL (system service) on `localhost:5432`, DB/user/pw `test` | `DB_URI` env var; schema pre‑applied at Alembic head `32f25cbf12f6` (77 tables) |
| Cache / rate‑limit | Redis (system service) `redis://localhost` | `tests/test.env:78` `MEM_STORE_URI` |

**Exact commands used** (run from the baked repo root `/app` inside the container):

```bash
# 1. bring up the backing services (system services in the canonical image)
service postgresql start && service redis-server start

# 2. schema is already at Alembic head in the image; the canonical migration command is a no-op:
CONFIG=tests/test.env alembic upgrade head          # .github/workflows/main.yml:98

# 3. run the temporary observation script against the real handler, capturing stdout+stderr
cd /app
export PYTHONPATH=/app
export DB_URI="postgresql://test:test@localhost:5432/test"
export GITHUB_ACTIONS_TEST=true
export DKIM_PRIVATE_KEY_PATH=/tmp/sl_investigate/dkim.key   # see "DKIM key note" below
/app/venv/bin/python /tmp/sl_investigate/observe.py > /tmp/sl_investigate/run.log 2>&1
```

The observation script set `CONFIG` itself before importing any app module
(`os.environ["CONFIG"] = os.path.abspath("tests/test.env")`), exactly as
`tests/conftest.py:8-10` does. Because `app/config.py:69` calls `load_dotenv(...)`
with its default `override=False`, the pre‑exported `DB_URI` (port **5432**, the
system Postgres) wins over the `15432` value in `tests/test.env:17` — this is the
same topology the test suite uses.

**Outbound mail was suppressed two ways**, so nothing left the process:
`NOT_SEND_EMAIL=true` (`app/config.py:91`) and the in‑process capture hook
`mail_sender.store_emails_instead_of_sending()` (`app/mail_sender.py:102`), which
makes the real `send()` (`app/mail_sender.py:126`) retain the fully transformed
outgoing message in `mail_sender.get_stored_emails()` (`app/mail_sender.py:108`)
instead of transmitting it. The `From` (Q3) and `Message-ID` (Q2) values quoted
below are read from that retained `EmailMessage` — i.e. the exact bytes the code
produced.

**DKIM key note (environment fix, no answer values affected).** The forward path
DKIM‑signs the outgoing message (`email_handler.py:891` → `app/email_utils.py:457`
`add_dkim_signature`). `dkimpy 1.0.5` requires a **PKCS#1** RSA key
(`-----BEGIN RSA PRIVATE KEY-----`), but the image's `local_data/dkim.key` and the
key produced by this image's OpenSSL 3.0.16 `genrsa` are **PKCS#8**
(`-----BEGIN PRIVATE KEY-----`); with the PKCS#8 key the canonical test
`tests/test_email_handler.py::test_preserve_headers` fails with
`Exception: Cannot create DKIM signature`. The **same key material** was therefore
re‑encoded to PKCS#1 at a temp path and pointed to via `DKIM_PRIVATE_KEY_PATH`:

```bash
openssl rsa -in /app/local_data/dkim.key -traditional -out /tmp/sl_investigate/dkim.key
```

`local_data/dkim.key` was **not** modified. After this fix the canonical test passes
(`1 passed`), and the forward completes to `E200`. The DKIM signature is not one of
the four questions and its value does not affect the `From` header, the `Message-ID`,
the log text, or the database records reported here.

### How the real path was exercised

The blessed way to drive the production handler without a live SMTP socket (used by
`tests/test_email_handler.py`) is to build an `aiosmtpd` `Envelope` + a standard
`EmailMessage` and call `email_handler.handle(envelope, msg)` (`email_handler.py:1945`).
Inside `handle()`, each recipient is routed (`email_handler.py:2195-2213`): a
reverse‑alias recipient → **reply** via `handle_reply` (`email_handler.py:966`), any
other recipient → **forward** via `handle_forward` (`email_handler.py:536`). The
temporary script seeded one `User` + default `Mailbox` (`tests/utils.py:17`
`create_new_user`) and one random alias `@sl.local` (`app/models.py:1721`
`Alias.create_new_random`), then exercised: **forward ×2**, **failure** (non‑existent
alias), and **reply ×2**. The script lived at `/tmp/sl_investigate/observe.py`
(outside the repository) and was deleted afterward.

The `SL` logger writes every line to stdout at `DEBUG` (`app/log.py:41,51`, logger
name `SL` at `app/log.py:79`, timestamps in **UTC** via `time.gmtime`
`app/log.py:43`) using this exact format string (`app/log.py:12-15`):

```python
_log_format = (
    "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
    '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
)
```

Because the format embeds `"%(pathname)s:%(lineno)d"`, **every captured log line
below is self‑citing** — it carries the emitting `file:line` produced by the running
code. (The `%(message_id)s` slot is the Postfix queue id injected by
`app/log.py:22` `set_message_id`; it is empty here because these messages were not
delivered by Postfix, shown as the blank ` -  - ` between `funcName()` and the message.)

> A note on line numbers: the emitting line number in each log line is the line of
> the `LOG.d(...)` **call**. For multi‑line calls this is one less than the line of
> the message string literal (e.g. the "Forward phase" call is at `email_handler.py:2202`
> while the string literal is at `:2203`; "alias … not exist" call at `:545`, literal
> at `:546`). This document cites the **observed** call‑site number that actually
> appears in the output.

---

## Q1 — Exact log text: successful forward vs. non‑existent‑alias failure

**Question:** *"what is the exact log message text that appears when an email is
successfully forwarded versus when it fails due to a non‑existent alias."*

### Q1a — Successful forward (returns `E200`)

Command that produced it (forward of a plain message from a random external sender
to the seeded alias `word_word485@sl.local`):

```
[FWD1] INPUT from=shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com to=word_word485@sl.local orig_mid=<orig-1-271@sender.example.com>
```

**Unedited captured `SL` log block for the successful forward** (from
`/tmp/sl_investigate/run.log`):

```text
2026-07-08 21:26:45,004 - SL - DEBUG - 271 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 21:26:45,006 - SL - DEBUG - 271 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com, rcpt_tos:['word_word485@sl.local'], header_from:shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com, header_to:word_word485@sl.local, cc:None, reply-to:None, message_id:<orig-1-271@sender.example.com>, client_ip:None, headers:[('From', 'shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com'), ('To', 'word_word485@sl.local'), ('Subject', 'Forward test 1'), ('Message-ID', '<orig-1-271@sender.example.com>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-08 21:26:45,010 - SL - DEBUG - 271 - "/app/email_handler.py:2202" - handle() -  - Forward phase shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com(shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com) -> word_word485@sl.local
2026-07-08 21:26:45,017 - SL - DEBUG - 271 - "/app/email_handler.py:580" - handle_forward() -  - Create or get contact for from_header:shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com
2026-07-08 21:26:45,035 - SL - DEBUG - 271 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 11 shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com 22> for alias <Alias 22 word_word485@sl.local> with email shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com invalid_email=False
2026-07-08 21:26:45,035 - SL - INFO - 271 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() -  - DMARC check disabled
2026-07-08 21:26:45,043 - SL - DEBUG - 271 - "/app/email_handler.py:688" - forward_email_to_mailbox() -  - Forward <Contact 11 shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com 22> -> <Alias 22 word_word485@sl.local> -> <Mailbox 11 user_h3w3jiveah@mailbox.test>
2026-07-08 21:26:45,046 - SL - DEBUG - 271 - "/app/email_handler.py:740" - forward_email_to_mailbox() -  - Create <EmailLog 11> for <Contact 11 shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com 22>, <User 11 Test User user_h3w3jiveah@mailbox.test>, <Mailbox 11 user_h3w3jiveah@mailbox.test>
2026-07-08 21:26:45,052 - SL - WARNING - 271 - "/app/email_handler.py:857" - forward_email_to_mailbox() -  - missing date header, create one
2026-07-08 21:26:45,052 - SL - DEBUG - 271 - "/app/email_handler.py:867" - forward_email_to_mailbox() -  - From header, new:"shsqfuyqjyzijvprjvxa at shsqfuyqjyzijvprjvxa.com" <shsqfuyqjyzijvprjvxa_at_shsqfuyqjyzijvprjvxa_com_ftavrpsjgv@sl.local>, old:shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com
2026-07-08 21:26:45,053 - SL - DEBUG - 271 - "/app/email_handler.py:316" - replace_header_when_forward() -  - Delete Cc header, old value None
2026-07-08 21:26:45,053 - SL - DEBUG - 271 - "/app/email_handler.py:313" - replace_header_when_forward() -  - Replace To header, old: word_word485@sl.local, new: word_word485@sl.local
2026-07-08 21:26:45,053 - SL - INFO - 271 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() -  - Email has no unsubscribe header
2026-07-08 21:26:45,062 - SL - DEBUG - 271 - "/app/email_handler.py:893" - forward_email_to_mailbox() -  - Forward mail from shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com to user_h3w3jiveah@mailbox.test, mail_options:[], rcpt_options:[] 
2026-07-08 21:26:45,063 - SL - DEBUG - 271 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Forward test 1', from '"shsqfuyqjyzijvprjvxa at shsqfuyqjyzijvprjvxa.com" <shsqfuyqjyzijvprjvxa_at_shsqfuyqjyzijvprjvxa_com_ftavrpsjgv@sl.local>' to 'word_word485@sl.local'
```

Return value of `handle()` (the SMTP status a real sender would receive):

```
[FWD1] handle() returned: '250 Message accepted for delivery'
```

**The notable success lines and their emitting `file:line`:**

| Log message (verbatim) | Emitting `file:line` |
|------------------------|----------------------|
| `Forward phase %s(%s) -> %s` | `email_handler.py:2202` (dispatch; literal at `:2203`) |
| `Create or get contact for from_header:%s` | `email_handler.py:580` |
| `Created contact <Contact …> for alias <Alias …> with email … invalid_email=%s` | `app/contact_utils.py:110` |
| `Forward %s -> %s -> %s` (contact → alias → mailbox) | `email_handler.py:688` |
| `Create %s for %s, %s, %s` (the `EmailLog`) | `email_handler.py:740` |
| `From header, new:%s, old:%s` | `email_handler.py:867` |
| `Forward mail from %s to %s, mail_options:%s, rcpt_options:%s ` | `email_handler.py:893` (literal at `:894`) |
| `send email with subject '%s', from '%s' to '%s'` | `app/mail_sender.py:131` (literal at `:132`) |
| returned `250 Message accepted for delivery` | `app/email/status.py:2` `E200` |

**Cause → effect.** The recipient `word_word485@sl.local` matches the seeded alias, so
`handle()` routes to `handle_forward` (`email_handler.py:536`), which creates/looks‑up
the sender `Contact`, persists an `EmailLog`, rewrites the `From` header, DKIM‑signs,
and calls `send()`. Because at least one delivery succeeds, the status‑aggregation at
`email_handler.py:2227-2231` returns that delivery's status — `status.E200`
(`"250 Message accepted for delivery"`, `app/email/status.py:2`).

### Q1b — Non‑existent alias failure (returns `E515`)

To reliably reach the failure branch, the message was addressed to a syntactically
valid but non‑existent address at `sl.local` with **no directory separator**, so
on‑the‑fly creation cannot rescue it.

Command that produced it:

```
[FAIL] INPUT from=jiewrdrzuqvewairqatd@jiewrdrzuqvewairqatd.com to=nonexistent-wfbdxhluzojpgwcupijh@sl.local orig_mid=<fail-271@sender.example.com>
```

**Unedited captured `SL` log block for the failure** (from `/tmp/sl_investigate/run.log`):

```text
2026-07-08 21:26:45,130 - SL - DEBUG - 271 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 21:26:45,131 - SL - DEBUG - 271 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:jiewrdrzuqvewairqatd@jiewrdrzuqvewairqatd.com, rcpt_tos:['nonexistent-wfbdxhluzojpgwcupijh@sl.local'], header_from:jiewrdrzuqvewairqatd@jiewrdrzuqvewairqatd.com, header_to:nonexistent-wfbdxhluzojpgwcupijh@sl.local, cc:None, reply-to:None, message_id:<fail-271@sender.example.com>, client_ip:None, headers:[('From', 'jiewrdrzuqvewairqatd@jiewrdrzuqvewairqatd.com'), ('To', 'nonexistent-wfbdxhluzojpgwcupijh@sl.local'), ('Subject', 'Failure test'), ('Message-ID', '<fail-271@sender.example.com>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-08 21:26:45,134 - SL - DEBUG - 271 - "/app/email_handler.py:2202" - handle() -  - Forward phase jiewrdrzuqvewairqatd@jiewrdrzuqvewairqatd.com(jiewrdrzuqvewairqatd@jiewrdrzuqvewairqatd.com) -> nonexistent-wfbdxhluzojpgwcupijh@sl.local
2026-07-08 21:26:45,141 - SL - DEBUG - 271 - "/app/email_handler.py:545" - handle_forward() -  - alias nonexistent-wfbdxhluzojpgwcupijh@sl.local not exist. Try to see if it can be created on the fly
2026-07-08 21:26:45,146 - SL - INFO - 271 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() -  - Cannot auto-create custom domain alias for nonexistent-wfbdxhluzojpgwcupijh@sl.local because there's no custom domain for sl.local
2026-07-08 21:26:45,146 - SL - INFO - 271 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() -  - Cannot auto-create nonexistent-wfbdxhluzojpgwcupijh@sl.local since it has no directory separator
2026-07-08 21:26:45,146 - SL - DEBUG - 271 - "/app/email_handler.py:551" - handle_forward() -  - alias nonexistent-wfbdxhluzojpgwcupijh@sl.local cannot be created on-the-fly, return 550
```

Return value of `handle()`:

```
[FAIL] handle() returned: '550 SL E515 Email not exist'
[FAIL] stored_count_after_failure=0
```

**The notable failure lines and their emitting `file:line`:**

| Log message (verbatim) | Emitting `file:line` |
|------------------------|----------------------|
| `alias %s not exist. Try to see if it can be created on the fly` | `email_handler.py:545` (literal at `:546`) |
| `Cannot auto-create custom domain alias for … because there's no custom domain for sl.local` | `app/alias_utils.py:104` |
| `Cannot auto-create … since it has no directory separator` | `app/alias_utils.py:165` |
| `alias %s cannot be created on-the-fly, return 550` | `email_handler.py:551` |
| returned `550 SL E515 Email not exist` | `app/email/status.py:51` `E515` |

**Cause → effect.** No alias matches `nonexistent-wfbdxhluzojpgwcupijh@sl.local`, so
`handle_forward` logs "not exist" (`email_handler.py:545`) and calls `try_auto_create`
(`email_handler.py:549` → `app/alias_utils.py:202`). Auto‑creation requires either a
catch‑all custom domain or a directory address (`app/alias_utils.py:274` / `:227`); a
freshly seeded user has neither, so both checks decline (the two `alias_utils`
`INFO` lines) and `try_auto_create` returns `None`. `handle_forward` then logs the
"cannot be created on-the-fly" line (`email_handler.py:551`) and returns
`[(False, status.E515)]` (`email_handler.py:555`). Since every delivery failed, the
aggregation at `email_handler.py:2233` returns the first failure —
`status.E515` (`"550 SL E515 Email not exist"`, `app/email/status.py:51`). No message
is stored (`stored_count_after_failure=0`), confirming nothing was forwarded.

**The exact difference (success vs. failure).** A successful forward emits the
`Forward %s -> %s -> %s`, `Create %s …` (`EmailLog`), `From header, new:… old:…`, and
`Forward mail from … to …` `DEBUG` lines and returns `250 Message accepted for
delivery`. A non‑existent‑alias failure emits `alias … not exist …` followed by the
two `alias_utils` auto‑create refusals and `alias … cannot be created on-the-fly,
return 550`, and returns `550 SL E515 Email not exist`.

---

## Q2 — The SL `Message-ID` vs. the original `Message-ID`

**Question:** *"what specific SL Message-ID gets generated during forwarding and how
does it differ from the original Message-ID."*

**Key finding (the honest answer to the "specific SL Message-ID"):** during a **pure
forward, no SL Message-ID is generated at all** — the sender's original `Message-ID`
is *preserved* on the outgoing message and stored in `EmailLog.message_id`, while
`EmailLog.sl_message_id` stays `NULL`. A new **SL Message-ID is minted only in the
reply phase**, by `make_msgid(str(email_log.id), get_email_domain_part(alias.email))`
(`email_handler.py:1311-1313`). Both paths were therefore exercised and are shown
below.

### Q2a — Forward preserves the original `Message-ID`

The message was sent with `Message-ID: <orig-1-271@sender.example.com>`. The captured
outgoing (forwarded) message carries the **same** value, unchanged:

```
[FWD1] OUT[0] Message-ID = '<orig-1-271@sender.example.com>'
```

And the persisted `EmailLog` for that forward stores the original and leaves the SL
column empty:

```
[FWD1] EmailLog id=11 created_at=<Arrow [2026-07-08T21:26:45.044697+00:00]> is_reply=False message_id='<orig-1-271@sender.example.com>' sl_message_id=None MessageIDMatching_count=0
```

*Why:* `forward_email_to_mailbox` persists `EmailLog.create(..., message_id=str(msg[headers.MESSAGE_ID]), ...)`
(`email_handler.py:737`); the forward keeps the `Message-ID` header (it only rewrites
`In-Reply-To`/`References` from SL values back to originals via
`replace_sl_message_id_by_original_message_id`, `email_handler.py:931`). Hence the
forwarded `Message-ID` equals the sender's original and `sl_message_id` is `NULL`.

### Q2b — Reply mints the SL `Message-ID`

When the mailbox replies to the reverse‑alias, `handle_reply` calls
`replace_original_message_id` (`email_handler.py:1296`), which mints the SL id and
logs it (`email_handler.py:1314`):

```text
2026-07-08 21:26:45,168 - SL - DEBUG - 271 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178354600516.271.7847110207061318470.13@sl.local>
```

The captured outgoing (relayed) reply message carries that minted id, and a
`MessageIDMatching` row records the mapping:

```
[REPLY1] OUT[0] Message-ID = '<178354600516.271.7847110207061318470.13@sl.local>'
[REPLY1] MessageIDMatching id=1 created_at=<Arrow [2026-07-08T21:26:45.168684+00:00]> sl_message_id='<178354600516.271.7847110207061318470.13@sl.local>' original_message_id='<reply-orig-1-271@mailbox.test>' email_log_id=13
```

The `sl_message_id` column transition (`NULL` after forward → set after reply) is
visible in the final DB dump — forward logs (`is_reply=False`) keep `sl_message_id=None`,
reply logs (`is_reply=True`) have it populated:

```
[FINAL] EmailLog id=11 created_at=<Arrow [2026-07-08T21:26:45.044697+00:00]> is_reply=False message_id='<orig-1-271@sender.example.com>' sl_message_id=None MessageIDMatching=[]
[FINAL] EmailLog id=13 created_at=<Arrow [2026-07-08T21:26:45.157998+00:00]> is_reply=True message_id='<reply-orig-1-271@mailbox.test>' sl_message_id='<178354600516.271.7847110207061318470.13@sl.local>' MessageIDMatching=[(1, '<178354600516.271.7847110207061318470.13@sl.local>', '<reply-orig-1-271@mailbox.test>')]
```

### The structural difference

`make_msgid(idstring, domain)` (Python `email.utils`, imported at
`email_handler.py:42`) produces `<{int(time*100)}.{pid}.{randbits}.{idstring}@{domain}>`.
Here `idstring = str(email_log.id)` and `domain = get_email_domain_part(alias.email)`
(`app/email_utils.py:448`) = the **alias/SL domain** `sl.local`. Decomposing the
observed SL id `<178354600516.271.7847110207061318470.13@sl.local>`:

| Component | Observed value | Meaning |
|-----------|----------------|---------|
| timestamp | `178354600516` | `int(time.time()*100)` at mint time |
| pid | `271` | the process id (matches the `- 271 -` field in every log line of that run) |
| random | `7847110207061318470` | 64‑bit random component from `make_msgid` |
| **idstring** | `13` | **`str(email_log.id)`** — the reply's `EmailLog.id` (13) |
| domain | `sl.local` | the alias/SL domain, not the sender's |

Contrast with the original `Message-ID` (`<orig-1-271@sender.example.com>`): an
**arbitrary local‑part at the sender's domain**, entirely chosen by the sending MTA.
So the two differ in **who owns the domain** (sender's domain vs. `sl.local`), in
**structure** (free‑form vs. the fixed four‑token `time.pid.rand.emaillog_id` shape),
and in the fact that the SL id **embeds the SimpleLogin `EmailLog.id`** as its last
token. The second reply run confirms the `email_log.id` embedding is not coincidental —
its id token tracks the reply `EmailLog`:

```
[FINAL] EmailLog id=14 created_at=<Arrow [2026-07-08T21:26:45.196192+00:00]> is_reply=True message_id='<reply-orig-2-271@mailbox.test>' sl_message_id='<178354600520.271.11843380804821950715.14@sl.local>' MessageIDMatching=[(2, '<178354600520.271.11843380804821950715.14@sl.local>', '<reply-orig-2-271@mailbox.test>')]
```

(EmailLog `id=14` → SL id ends in `.14@sl.local`.) See *Run‑to‑run variability* for
which parts of the SL id are stable vs. varying.

---

## Q3 — The transformed `From` header (including the reverse‑alias `reply_email` format)

**Question:** *"what is the exact From header value in the forwarded email after
transformation including the reply-email address format."*

The forward replaces the sender's `From` with `contact.new_addr()`
(`email_handler.py:864-867`; `Contact.new_addr` at `app/models.py:2008`). **The exact,
unedited transformed `From` value captured on the outgoing forwarded message** (run 1,
forward 1):

```
[FWD1] OUT[0] From = '"shsqfuyqjyzijvprjvxa at shsqfuyqjyzijvprjvxa.com" <shsqfuyqjyzijvprjvxa_at_shsqfuyqjyzijvprjvxa_com_ftavrpsjgv@sl.local>'
```

The same value appears in the emitting log line (`email_handler.py:867`):

```text
2026-07-08 21:26:45,052 - SL - DEBUG - 271 - "/app/email_handler.py:867" - forward_email_to_mailbox() -  - From header, new:"shsqfuyqjyzijvprjvxa at shsqfuyqjyzijvprjvxa.com" <shsqfuyqjyzijvprjvxa_at_shsqfuyqjyzijvprjvxa_com_ftavrpsjgv@sl.local>, old:shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com
```

### Anatomy of the value

The `From` is a display‑name + reverse‑alias address, RFC‑2047 encoded via
`sl_formataddr` (`app/models.py:2044-2045`; def `app/email_utils.py:1501`):

- **Display name — `"shsqfuyqjyzijvprjvxa at shsqfuyqjyzijvprjvxa.com"`.** This is the
  **default `sender_format = AT`** rendering. `Contact.new_addr` reads
  `sender_format = user.sender_format if user else SenderFormatEnum.AT.value`
  (`app/models.py:2019`); the seeded user has `sender_format=0` (observed
  `[SEED] … user.sender_format=0`), and `SenderFormatEnum.AT = 0` (`app/models.py:204`,
  column default `"0"` at `app/models.py:416`). The AT branch (`app/models.py:2028-2034`)
  builds the display name by replacing `@` with `" at "` in the contact's
  `website_email`; because this contact has no distinct display name (`name=None`,
  see Q4), the name is exactly the formatted email `shsqfuyqjyzijvprjvxa at shsqfuyqjyzijvprjvxa.com`.
- **Address — `shsqfuyqjyzijvprjvxa_at_shsqfuyqjyzijvprjvxa_com_ftavrpsjgv@sl.local`.**
  This is the contact's `reply_email` (the reverse‑alias), from
  `generate_reply_email(contact_email, alias)` (`app/email_utils.py:1103`).

### Observed `reply_email` format and why it takes the sender‑inclusive branch

`generate_reply_email` has two branches (`app/email_utils.py:1137-1148`). The observed
value takes the **sender‑inclusive** branch, **not** the "20–50 random characters"
default:

```
reply_email = f"{contact_email}_{random_string(random_length)}@{reply_domain}"   # app/email_utils.py:1142, random_length = randint(5, 10)
```

*Why (verified at runtime, not assumed):* the branch is gated by
`user.include_sender_in_reverse_alias` (`app/email_utils.py:1116-1117`). This column
has an **ORM‑side `default=True`** (`app/models.py:455-457`
`include_sender_in_reverse_alias = sa.Column(sa.Boolean, default=True, nullable=False, server_default="0")`).
The canonical seeding helper `create_new_user` creates the user through the ORM
(`User.create(...)`, `tests/utils.py:23`), so the ORM default `True` applies. A direct
DB read confirmed it for the seeded user:

```
id | email                        | include_sender_in_reverse_alias | sender_format
11 | user_h3w3jiveah@mailbox.test | t                               | 0
```

Consequently `contact_email` is sanitized (`app/email_utils.py:1120-1126`:
`convert_to_id` → `sanitize_email` → truncate to 45 chars → `@`→`_at_` → `.`→`_` →
`convert_to_alphanumeric`) and used as a prefix. For
`shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com` this yields
`shsqfuyqjyzijvprjvxa_at_shsqfuyqjyzijvprjvxa_com`, followed by `_` + a random token of
length `randint(5, 10)` (`app/email_utils.py:1138`, `random_string` at `app/utils.py:41`
= lowercase letters), at `reply_domain = config.EMAIL_DOMAIN = sl.local`
(`app/email_utils.py:1129`).

> If a user instead had `include_sender_in_reverse_alias = False`, the code takes the
> default branch (`app/email_utils.py:1145-1148`)
> `reply_email = f"{random_string(random_length)}@{reply_domain}"` with
> `random_length = randint(20, 50)` — i.e. `<20–50 random lowercase chars>@sl.local`,
> with **no** sender prefix. That branch was **not** exercised here because the
> canonical seeded user has the flag `True`.

**Confirmation across additional forwards** (same structure, different random token —
these are the reverse‑aliases, and their random suffixes are a legitimate source of
run‑to‑run variation):

```
[FWD2] OUT[0] From = '"kpmlzogtkdhxjagptyer at kpmlzogtkdhxjagptyer.com" <kpmlzogtkdhxjagptyer_at_kpmlzogtkdhxjagptyer_com_hywzri@sl.local>'
# second independent process (run2), forward 1:
[FWD1] OUT[0] From = '"ecuyrsmssqvviulnmbfk at ecuyrsmssqvviulnmbfk.com" <ecuyrsmssqvviulnmbfk_at_ecuyrsmssqvviulnmbfk_com_lxarfzino@sl.local>'
```

**Cause → effect (and the SimpleLogin model).** The forward masks the real sender: the
outgoing `From` is a **reverse‑alias** address at `sl.local` (never the sender's real
address), decorated with a human‑readable "who this is from" display name in the
`AT` format. This matches SimpleLogin's documented reverse‑alias behavior — mail
arrives *from* a reverse‑alias, and replying to that reverse‑alias relays back to the
original sender with the user's real address masked — but the concrete address above
is the value produced by this run, not from documentation.

---

## Q4 — Database records created by a single forward (real IDs + timestamps)

**Question:** *"what database records are created during a single forward operation
show me the actual record IDs and timestamps."*

A single forward operation persists exactly **two** rows: one `Contact` (only when the
sender is new) and one `EmailLog`. **No `MessageIDMatching` row is created on the
forward path.** The actual rows created by the first forward (sender
`shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com` → alias `word_word485@sl.local`),
captured directly from the database:

```
[FWD1] Contact id=11 created_at=<Arrow [2026-07-08T21:26:45.027681+00:00]> website_email='shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com' reply_email='shsqfuyqjyzijvprjvxa_at_shsqfuyqjyzijvprjvxa_com_ftavrpsjgv@sl.local' name=None
[FWD1] EmailLog id=11 created_at=<Arrow [2026-07-08T21:26:45.044697+00:00]> is_reply=False message_id='<orig-1-271@sender.example.com>' sl_message_id=None MessageIDMatching_count=0
```

| Table | `id` | `created_at` (UTC) | Key columns |
|-------|------|--------------------|-------------|
| `contact` (`app/models.py:1863`) | **11** | **2026‑07‑08T21:26:45.027681+00:00** | `website_email=shsqfuyqjyzijvprjvxa@shsqfuyqjyzijvprjvxa.com`, `reply_email=shsqfuyqjyzijvprjvxa_at_shsqfuyqjyzijvprjvxa_com_ftavrpsjgv@sl.local`, `name=None` |
| `email_log` (`app/models.py:2060`) | **11** | **2026‑07‑08T21:26:45.044697+00:00** | `is_reply=False`, `message_id=<orig-1-271@sender.example.com>`, `sl_message_id=None` |
| `message_id_matching` (`app/models.py:3365`) | — | — | **none created on the forward path** (`MessageIDMatching_count=0`) |

**Shape of `id` / `created_at`.** Both tables inherit `ModelMixin` (`app/models.py:62-65`):
`id` is an autoincrement integer PK (`:63`), `created_at` is an `ArrowType` defaulting
to `arrow.utcnow` (`:64`, hence the `+00:00`/UTC offset), and `updated_at` (`:65`).
(`ModelMixin._repr_hide` at `:67` hides the timestamps from `repr()`, so the columns
were read directly rather than via `print(row)`.) The `id` values are **11**, not `1`,
because the image ships a pre‑migrated, pre‑seeded database (77 tables at Alembic head);
these are the genuine autoincrement values observed (`user.id=11`, `alias.id=22`).

**Order of creation (timestamps).** The `Contact` (`.027681`) precedes the `EmailLog`
(`.044697`), because `get_or_create_contact` (`email_handler.py:180` →
`app/contact_utils.py:42` `create_contact`; `reply_email = generate_reply_email(...)`
at `:89`; `Contact.create(...)` at `:92`) runs before `EmailLog.create(...)`
(`email_handler.py:732`). `EmailLog.message_id` / `sl_message_id` are `deferred`
columns (`app/models.py:2114`, `:2116`), read within the app context.

**No `MessageIDMatching` on forward, one per reply.** The forward's
`MessageIDMatching_count=0` above confirms absence. A `MessageIDMatching` row
(`app/models.py:3365`; `sl_message_id` unique `:3371`, `original_message_id` unique
`:3372`, `email_log_id` FK) appears **only** on the reply path, created at
`email_handler.py:1316`. The final DB dump shows the two forward `EmailLog`s carry an
empty `MessageIDMatching` list while the two reply `EmailLog`s each carry exactly one:

```
[FINAL] EmailLog id=11 created_at=<Arrow [2026-07-08T21:26:45.044697+00:00]> is_reply=False message_id='<orig-1-271@sender.example.com>' sl_message_id=None MessageIDMatching=[]
[FINAL] EmailLog id=12 created_at=<Arrow [2026-07-08T21:26:45.107062+00:00]> is_reply=False message_id='<orig-2-271@sender.example.com>' sl_message_id=None MessageIDMatching=[]
[FINAL] EmailLog id=13 created_at=<Arrow [2026-07-08T21:26:45.157998+00:00]> is_reply=True message_id='<reply-orig-1-271@mailbox.test>' sl_message_id='<178354600516.271.7847110207061318470.13@sl.local>' MessageIDMatching=[(1, '<178354600516.271.7847110207061318470.13@sl.local>', '<reply-orig-1-271@mailbox.test>')]
[FINAL] EmailLog id=14 created_at=<Arrow [2026-07-08T21:26:45.196192+00:00]> is_reply=True message_id='<reply-orig-2-271@mailbox.test>' sl_message_id='<178354600520.271.11843380804821950715.14@sl.local>' MessageIDMatching=[(2, '<178354600520.271.11843380804821950715.14@sl.local>', '<reply-orig-2-271@mailbox.test>')]
```

(The second forward, for a different new sender, analogously created `Contact id=12`
and `EmailLog id=12`.)


---

## Run‑to‑run variability (this is the reported "inconsistency")

The identical input was executed in **two independent processes** (PID `271` →
`run.log`, PID `309` → `run2.log`). Comparing them isolates what is stable from what
varies. The variation is **by design** — it comes from `random_string`
(`app/utils.py:41`), `make_msgid` (`email_handler.py:1311`), autoincrement primary
keys (`app/models.py:63`), and `created_at = arrow.utcnow` (`app/models.py:64`) — and
is **not** a bug.

| Aspect | Run 1 (PID 271) | Run 2 (PID 309) | Stable or Varying |
|--------|-----------------|-----------------|-------------------|
| Forward success status | `250 Message accepted for delivery` | `250 Message accepted for delivery` | **Stable** (`E200`) |
| Failure status | `550 SL E515 Email not exist` | `550 SL E515 Email not exist` | **Stable** (`E515`) |
| Emitting `file:line` of key log lines | `:2202`, `:545`, `:551`, `:867`, `:1314` | `:2202`, `:545`, `:551`, `:867`, `:1314` | **Stable** |
| Log message templates | `Forward phase …`, `From header, new:… old:…`, `create a new sl_message_id …` | identical templates | **Stable** |
| Forward `Message-ID` (preserved) | `<orig-1-271@sender.example.com>` | `<orig-1-309@sender.example.com>` | Varying value, **stable rule** (equals the sender's original) |
| Transformed `From` structure | `"… at …" <…_at_…_com_<rand>@sl.local>` | `"… at …" <…_at_…_com_<rand>@sl.local>` | **Stable** structure |
| Reverse‑alias random token | `…_ftavrpsjgv@sl.local` | `…_lxarfzino@sl.local` | **Varying** (`random_string`, len 5–10) |
| SL `Message-ID` (reply 1) | `<178354600516.271.7847110207061318470.13@sl.local>` | `<178354615960.309.13781979540571388296.17@sl.local>` | structure **stable**, components **varying** |
| — its `pid` token | `271` | `309` | **Varying** (process id) |
| — its timestamp token | `178354600516` | `178354615960` | **Varying** (wall clock) |
| — its random token | `7847110207061318470` | `13781979540571388296` | **Varying** (`make_msgid`) |
| — its `email_log.id` token | `13` | `17` | **Varying** value, **stable rule** (= reply `EmailLog.id`) |
| — its domain | `sl.local` | `sl.local` | **Stable** (alias/SL domain) |
| Autoincrement ids (Contact / fwd EmailLog / reply EmailLog / MessageIDMatching) | `11 / 11 / 13 / 1` | `13 / 15 / 17 / 3` | **Varying** (autoincrement) |
| `created_at` timestamps | `…T21:26:45.*` | `…T21:29:19.*` | **Varying** (`arrow.utcnow`) |

**Summary of the "inconsistency."** Nothing about the *behavior* is inconsistent: the
forward always returns `E200`, the non‑existent alias always returns `E515`, the
forward always preserves the original `Message-ID`, the reply always mints an SL id of
the fixed four‑token shape at `@sl.local`, and the `From` is always the reverse‑alias
in the `AT` format. What differs every run are the **values inside those fixed
structures** — the reverse‑alias random suffix, the `make_msgid` timestamp/pid/random,
the embedded autoincrement `email_log.id`, and the row timestamps. Any monitoring or
test that pins these exact strings will perceive "inconsistency"; comparisons should
instead assert the stable structure (status codes, header shapes, domain, and the
`email_log.id`‑embedding rule).

---

## Cleanup & integrity

- The observation ran inside a disposable container started from the canonical image;
  the temporary script (`/tmp/sl_investigate/observe.py`), its logs, and the
  temporary PKCS#1 DKIM key (`/tmp/sl_investigate/dkim.key`) live **outside** the
  repository and were removed after capture. The container and its Postgres/Redis were
  torn down.
- **No SimpleLogin source, test, configuration, or migration file was modified**; in
  particular `local_data/dkim.key` was left untouched (the PKCS#1 re‑encoding was
  written to a temp path and selected via the `DKIM_PRIVATE_KEY_PATH` env var only).
- The only change to the repository is this new, untracked documentation file,
  `blitzy/documentation/app_2cd6ee777f8c.md`. `git status` reports a clean working
  tree apart from the new `blitzy/` directory.

### Appendix — `file:line` references cited (verified at commit `2cd6ee777f8c…`)

- Entry point / routing / aggregation: `email_handler.py:1945` (`handle`), `:2196`/`:2202`
  (Reply/Forward phase dispatch), `:2227-2233` (status aggregation).
- Forward: `email_handler.py:536` (`handle_forward`), `:545`/`:551`/`:555` (not‑exist →
  `E515`), `:679` (`forward_email_to_mailbox`), `:688`, `:732`/`:737` (`EmailLog.create`,
  `message_id`), `:740`, `:864-867` (`From` rewrite), `:893`, `:931`
  (`replace_sl_message_id_by_original_message_id`).
- Reply / SL Message‑ID: `email_handler.py:966` (`handle_reply`), `:1296`
  (`replace_original_message_id`), `:1311-1313` (`make_msgid`), `:1314`, `:1316`
  (`MessageIDMatching.create`), `:1338-1341`; `:42` (`make_msgid` import);
  `app/email_utils.py:448` (`get_email_domain_part`).
- Models: `app/models.py:62-65` (`ModelMixin` id/created_at/updated_at), `:67`
  (`_repr_hide`), `:204`/`:416` (`SenderFormatEnum.AT=0`, default), `:455-457`
  (`include_sender_in_reverse_alias` default `True`), `:1863` (`Contact`), `:2008`
  (`Contact.new_addr`, AT branch `:2028-2034`, `sl_formataddr` `:2045`), `:2060`
  (`EmailLog`; deferred `message_id` `:2114`, `sl_message_id` `:2116`), `:3365`
  (`MessageIDMatching`; `:3371`, `:3372`).
- Reverse‑alias: `app/email_utils.py:1103` (`generate_reply_email`), `:1116-1117`
  (flag read), `:1120-1126` (sanitize), `:1129` (`reply_domain`), `:1137-1143`
  (sender‑inclusive, `:1142`), `:1145-1148` (default), `:1501` (`sl_formataddr`);
  `app/utils.py:41` (`random_string`).
- Logging / status / capture: `app/log.py:12-15` (format), `:22`, `:41-43`, `:74-77`,
  `:79`; `app/email/status.py:2` (`E200`), `:51` (`E515`); `app/mail_sender.py:102`
  (`store_emails_instead_of_sending`), `:105`, `:108`, `:126`, `:131`, `:203`.
- Contacts / auto‑create: `email_handler.py:180` (`get_or_create_contact`),
  `app/contact_utils.py:42`/`:89`/`:92` (`create_contact`); `app/alias_utils.py:202`
  (`try_auto_create`), `:227` (directory), `:274` (custom domain).
- Seeding / config: `tests/utils.py:17` (`create_new_user`), `:90` (`random_email`);
  `app/models.py:1721` (`Alias.create_new_random`); `app/config.py:69` (`load_dotenv`,
  `override=False`), `:91`/`:92` (`NOT_SEND_EMAIL`/`EMAIL_DOMAIN`), `:192` (`DB_URI`);
  `tests/test.env:7`/`:8`/`:17`.

