# How SimpleLogin resolves an inbound reply to a Contact & forwarding destination — and how it can misroute to the wrong user

> **Investigation type:** read-only root-cause / Q&A, written from **observed runtime behavior**.
> Every behavioral claim below is backed by (a) the exact command, (b) the complete unedited output, and (c) a `file:line` citation.
> The repository was left **byte-for-byte unchanged**; all observation scripts lived in the container's `/tmp` (not the repo tree). See [Read-only guarantee](#read-only-guarantee-repository-left-unchanged).

---

## Direct answer / TL;DR

- **How the reply address is derived (R1):** the reply address is the **SMTP envelope recipient**. Inside `handle_reply` it is taken verbatim as `reply_email = rcpt_to` [`email_handler.py`:L972], then rewritten by `normalize_reply_email()` [`email_handler.py`:L984]. The message reaches `handle_reply` only because `is_reverse_alias(rcpt_to)` is true [`email_handler.py`:L2195 → `app/email_utils.py`:L1156].
- **How the Contact is identified (R2):** a single lookup `contact = Contact.get_by(reply_email=reply_email)` [`email_handler.py`:L986], which is `Session.query(Contact).filter_by(reply_email=...).first()` [`app/models.py`:L83-L84] — an **unordered `LIMIT 1`** (the emitted SQL has **no `ORDER BY`**).
- **The forwarding destination (R3):** `alias = contact.alias` [L994] → `user = alias.user` [L1004] → authorized `mailbox = get_mailbox_from_mail_from(mail_from, alias)` [L1019/L1364] → the message is relayed to `contact.website_email` [~L1226].
- **Across many identical replies (R4):** resolution is **stable/deterministic when `reply_email` is unique** — 12/12 events resolved to the same Contact in each of two separate process runs.
- **Can the same reply email resolve to different Contacts over time? (R5):** **Yes — once a duplicate `reply_email` exists.** The `reply_email` column is a **non-unique** index, so two Contacts can share a `reply_email`; the unordered `.first()` then returns a **database-arbitrary** matching row. Observed live: the *same* reply address resolved to Contact 327 (user 904) and, after an unrelated write reordered the heap, to Contact 328 (user 905).
- **Can resolution fail temporarily? (R6):** **Yes.** A syntactically-valid reverse-alias with no matching Contact returns `E502` [L987-L989]; this is *temporary* (it resolves as soon as the Contact exists). A separate *normalization asymmetry* can make a stored Contact permanently unreachable via its own address (also `E502`), and a bad reply domain yields `E501` [L977-L981].
- **Correct-now-wrong-later? (R7):** A mis-resolution **usually bounces with `E214`** because `get_mailbox_from_mail_from` rejects a sender that is not authorized for the *resolved* alias. But it **silently delivers to the wrong user** when the resolved alias has `disable_email_spoofing_check=True` [L1021-L1029], or delivers to the wrong contact when a mailbox is shared/authorized so `mail_from` matches.
- **Root cause (R8/R9):** reply routing **assumes `reply_email` uniquely identifies one Contact**, but that uniqueness is enforced only **softly** (a check-then-act loop at generation time [`app/email_utils.py`:L1136-L1151 + `available_sl_email` `app/models.py`:L1425-L1432]) and by **no** DB constraint (`reply_email` is `index=True`, non-unique [`app/models.py`:L1899; migration `...78403c7b8089_.py`:L22]; the only unique constraint is `uq_contact(alias_id, website_email)` [`app/models.py`:L1874-L1876; migration `...0809266d08ca_.py`:L45]). The lookup is an unordered `LIMIT 1`. Therefore **if** a duplicate `reply_email` ever exists, the arbitrary row selection can pick the wrong Contact → wrong alias → wrong user; the anti-spoofing mailbox check is the mitigating control that usually converts this into an `E214` rejection rather than a silent misdelivery.
- **Calibrated risk:** a *natural* collision of the 20–50-char random local part [`app/email_utils.py`:L1145] is astronomically unlikely, so this is a **latent** uniqueness/timing defect — it fires when a duplicate `reply_email` is introduced administratively/programmatically (or by any future code path that inserts `reply_email` without the soft check) — **not** a routinely-triggered bug in normal operation.

---

## Environment & how it was run

All observations were produced on the **canonical stack** inside the provided Docker container `sl-app`
(image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), which bind-mounts the repository working tree to `/app`.
The container is **not** runnable with the host's system interpreter (Python 3.12 / SQLAlchemy 2.0 are incompatible with the pinned 1.3.24 API); the container provides Python **3.10** and SQLAlchemy **1.3.24** as pinned by `pyproject.toml`:L61 (`python = "^3.10"`), `pyproject.toml`:L116 (`SQLAlchemy = "1.3.24"`) and `Dockerfile`:L8 (`FROM python:3.10`).

Canonical run pattern used for every command below:

```
docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python <script>'
```

`/root/sl_env.sh` sets the canonical config: `CONFIG=/app/tests/test.env`, `EMAIL_DOMAIN=sl.local`, `NOT_SEND_EMAIL=true`, `DISABLE_RATE_LIMIT=1`, and `DB_URI=postgresql://test:test@localhost:5432/test` (PostgreSQL 15, running in the container).

**Environment verification (command + complete output):**

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && \
    echo "### python version ###" && /app/venv/bin/python --version && \
    echo "### sqlalchemy version ###" && /app/venv/bin/python -c "import sqlalchemy; print(sqlalchemy.__version__)" && \
    echo "### postgres reachability ###" && pg_isready -h localhost -p 5432 && \
    echo "### config.DB_URI (canonical) ###" && /app/venv/bin/python -c "from app import config; print(config.DB_URI)" 2>/dev/null | tail -1 && \
    echo "### import email_handler ###" && /app/venv/bin/python -c "import email_handler; print(\"handle =\", email_handler.handle)" 2>/dev/null | tail -1'
### python version ###
Python 3.10.18
### sqlalchemy version ###
1.3.24
### postgres reachability ###
localhost:5432 - accepting connections
### config.DB_URI (canonical) ###
postgresql://test:test@localhost:5432/test
### import email_handler ###
handle = <function handle at 0x799100922cb0>
```

```
$ cd /tmp/blitzy/app/blitzy-c2a41c33-7e7e-4397-9fc6-7fc76ebf8165_9eedb3 && \
    echo "### git HEAD ###" && git log --oneline -1 && \
    echo "### git branch (current) ###" && git rev-parse --abbrev-ref HEAD
### git HEAD ###
2cd6ee77 chore: emit some missing contact audit logs (#2269)
### git branch (current) ###
blitzy-c2a41c33-7e7e-4397-9fc6-7fc76ebf8165
```

The evidence chain is at HEAD **`2cd6ee77`**. (The `handle` function address varies per interpreter process; it is shown only to prove the module imported and `handle` is callable.)

---

## Methodology & instrumentation (no source edits)

Every behavioral claim is produced by driving the **real** `aiosmtpd`-style entry point
`email_handler.handle(envelope, msg)` [`email_handler.py`:L1945], which routes replies to
`handle_reply` [`email_handler.py`:L966]. Internal values were captured **without editing any tracked file**, using a
runtime monkeypatch (a *spy*) around `Contact.get_by`, plus the handler's own DEBUG log lines, plus direct DB queries
for before/during/after state. The shared preamble (kept only in the container's `/tmp/sl_investigation/`, never in the
repo) was, verbatim:

```python
"""
Shared instrumentation preamble for the SimpleLogin reply-resolution investigation.
Lives in the container's /tmp (NOT bind-mounted) so the repository stays byte-for-byte unchanged.

- Pushes a real Flask app context (canonical entry-point setup, mirrors tests/conftest.py).
- Leaves the app's own DEBUG logging intact (LOG.d/LOG.w/LOG.i print resolved values).
- Installs a non-invasive spy around app.models.Contact.get_by that RECORDS the lookup
  kwargs and the resolved Contact identity (id/alias_id/user_id), and OPTIONALLY prints
  each call, then delegates to the ORIGINAL classmethod. Runtime monkeypatch only; no
  source file is edited.
"""
import os
os.environ.setdefault("CONFIG", "/app/tests/test.env")

from server import create_app  # noqa: E402

app = create_app()
app.app_context().push()

import app.models as models  # noqa: E402
from app.models import Contact  # noqa: E402

_orig_contact_get_by = Contact.get_by  # bound classmethod captured before patch
SPY_ON = {"enabled": False}
SPY_PRINT = {"enabled": True}
SPY_LOG = []


def _fmt(result):
    if result is None:
        return "None"
    return (f"Contact(id={result.id}, alias_id={result.alias_id}, "
            f"user_id={result.user_id}, website_email={result.website_email!r})")


def _spy_get_by(**kw):
    result = _orig_contact_get_by(**kw)
    if SPY_ON["enabled"]:
        SPY_LOG.append((dict(kw), None if result is None else result.id))
        if SPY_PRINT["enabled"]:
            print(f"[SPY Contact.get_by] kwargs={kw} -> {_fmt(result)}", flush=True)
    return result


def install_spy():
    Contact.get_by = _spy_get_by


def enable_spy(print_calls=True):
    SPY_PRINT["enabled"] = print_calls
    SPY_ON["enabled"] = True


def disable_spy():
    SPY_ON["enabled"] = False


install_spy()
```

Notes on fidelity:
- The spy **delegates to the original** `Contact.get_by` and only records/prints; it changes no behavior. It fires for every `Contact.get_by(reply_email=...)` call site — routing `is_reverse_alias` [`app/email_utils.py`:L1158], the resolution in `handle_reply` [`email_handler.py`:L986], `available_sl_email` [`app/models.py`:L1428] and `create_contact` [`app/contact_utils.py`:L85/L118].
- Seed data uses the repository's own test helpers `create_new_user()` and `Alias.create_new_random()` from `tests/utils.py`.
- Outbound relay is captured with the repo's `mail_sender.store_emails_instead_of_sending()` / `get_stored_emails()`; for a reply, `SendRequest.envelope_to == contact.website_email`.
- Most experiment scripts print their own structured `[E<n>...]` observation lines *in addition to* the handler's own `LOG.d/LOG.w` lines; both are shown verbatim below.
- A few commands append `| grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"` — this removes only the **fixed startup banner** that the config loader prints on every interpreter start; it never removes experiment output. Where the banner matters it is shown (see the Environment section, which is unfiltered).
- Any value obtained by calling an internal function directly (rather than through `handle()`) is explicitly labelled **(direct-call illustration of the same lookup at `email_handler.py`:L986)**.
- Because the DB accumulates rows across runs, the row **ids differ between experiments** (e.g. E1 seeds Contact 324, E3 seeds 327/328); ids are always consistent *within* the output block that reports them.

---

## R1 — Reply-address derivation

**Direct answer:** The reply address is the **SMTP envelope recipient** (`rcpt_to`). In `handle_reply` it is assigned verbatim — `reply_email = rcpt_to` [`email_handler.py`:L972] — and then rewritten by `reply_email = normalize_reply_email(reply_email)` [`email_handler.py`:L984]. The message only reaches `handle_reply` because the router tests `if is_reverse_alias(rcpt_to):` [`email_handler.py`:L2195] → `handle_reply(envelope, copy_msg, rcpt_to)` [L2199]. `is_reverse_alias` [`app/email_utils.py`:L1156] returns true if a Contact already has that `reply_email` (`Contact.get_by(reply_email=address)` [L1158]) **or** the address is `@EMAIL_DOMAIN` and starts with `reply+`/`ra+` [L1161-L1163].

**Experiment E1 (also supports R2 and R3).** Seed `User1` + `Alias1` (+ default `Mailbox1`) + `Contact1` with a canonically-generated `reply_email` (`generate_reply_email`, `app/email_utils.py`:L1103), then drive a reply from the mailbox owner through the real `email_handler.handle`. The temporary driver (`/tmp/sl_investigation/e1_happy_path.py`) imports the `obs` spy, seeds via the repo's own test helpers, builds an `EmailMessage`, sets `envelope.mail_from` to the authorized mailbox and `envelope.rcpt_tos = [contact1.reply_email]`, and calls `email_handler.handle(envelope, msg)`.

Command + complete unedited output (startup banner filtered, as disclosed above):

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e1_happy_path.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
==============================================================================
E1 SEED (before): create User1 + Alias1(+default Mailbox1) + Contact1
==============================================================================
2026-07-06 23:15:41,864 - SL - INFO - 2969 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-06 23:15:41,877 - SL - DEBUG - 2969 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email swedes_smiles955@sl.local
2026-07-06 23:15:41,884 - SL - INFO - 2969 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
User1.id            = 901
User1.email         = user_c9zn2wnjoq@mailbox.test
Alias1.id           = 1564
Alias1.email        = swedes_smiles955@sl.local
Mailbox1 (default)  = id=1037 email=user_c9zn2wnjoq@mailbox.test
Contact1.id         = 324
Contact1.alias_id   = 1564
Contact1.user_id    = 901
Contact1.website_email = mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com
Contact1.reply_email   = mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local  (generate_reply_email, email_utils.py:L1103)

R1 derivation checks:
  is_reverse_alias(reply_email)  -> True   (email_utils.py:L1156 => handle_reply)
  normalize_reply_email(input)   -> 'mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local'   (email_validation.py:L25-38; identical for a clean key)

==============================================================================
E1 DRIVE (during): email_handler.handle(envelope, msg)  [email_handler.py:L1945]
  envelope.mail_from = user_c9zn2wnjoq@mailbox.test
  envelope.rcpt_tos  = ['mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local']
==============================================================================
2026-07-06 23:15:41,908 - SL - DEBUG - 2969 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-06 23:15:41,910 - SL - DEBUG - 2969 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_c9zn2wnjoq@mailbox.test, rcpt_tos:['mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local'], header_from:None, header_to:mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local, cc:None, reply-to:None, message_id:<e1-happy@investigation.local>, client_ip:None, headers:[('To', 'mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local'), ('Subject', 'E1 happy-path reply'), ('Message-ID', '<e1-happy@investigation.local>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[SPY Contact.get_by] kwargs={'reply_email': 'user_c9zn2wnjoq@mailbox.test'} -> None
[SPY Contact.get_by] kwargs={'reply_email': 'mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local'} -> Contact(id=324, alias_id=1564, user_id=901, website_email='mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com')
[SPY Contact.get_by] kwargs={'reply_email': 'mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local'} -> Contact(id=324, alias_id=1564, user_id=901, website_email='mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com')
2026-07-06 23:15:41,912 - SL - DEBUG - 2969 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_c9zn2wnjoq@mailbox.test(None) -> mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local
[SPY Contact.get_by] kwargs={'reply_email': 'mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local'} -> Contact(id=324, alias_id=1564, user_id=901, website_email='mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com')
2026-07-06 23:15:41,914 - SL - INFO - 2969 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-06 23:15:41,918 - SL - DEBUG - 2969 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 595> for <Contact 324 mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com 1564>, <User 901 Test User user_c9zn2wnjoq@mailbox.test>, <Mailbox 1037 user_c9zn2wnjoq@mailbox.test>
2026-07-06 23:15:41,924 - SL - DEBUG - 2969 - "/app/email_handler.py:1171" - handle_reply() -  - From header is swedes_smiles955@sl.local
[SPY Contact.get_by] kwargs={'reply_email': 'mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local'} -> Contact(id=324, alias_id=1564, user_id=901, website_email='mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com')
2026-07-06 23:15:41,925 - SL - DEBUG - 2969 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local, new: mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com
2026-07-06 23:15:41,925 - SL - DEBUG - 2969 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-06 23:15:41,927 - SL - DEBUG - 2969 - "/app/email_handler.py:1309" - replace_original_message_id() -  - reuse the sl_message_id <178337830542.2532.13222264564082674766.524@sl.local>
2026-07-06 23:15:41,929 - SL - WARNING - 2969 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-06 23:15:41,932 - SL - DEBUG - 2969 - "/app/email_handler.py:1212" - handle_reply() -  - send email from swedes_smiles955@sl.local to mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com, mail_options:[],rcpt_options:[]
2026-07-06 23:15:41,935 - SL - DEBUG - 2969 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'E1 happy-path reply', from 'swedes_smiles955@sl.local' to 'mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com'

==============================================================================
E1 RESULT (after):
==============================================================================
SMTP status returned by handle() = '250 Message accepted for delivery'
  status.E200 = '250 Message accepted for delivery'  (app/email/status.py:L2)
outbound messages captured = 1
  SendRequest.envelope_to = 'mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com'  (== Contact1.website_email? True)
  SendRequest.is_forward  = False
EmailLog rows for Contact1 = 1
  EmailLog(id=595, is_reply=True, alias_id=1564, user_id=901, mailbox_id=1037)

E1 DONE
```

**Reasoning:** `envelope.rcpt_tos = [contact1.reply_email]` is the only reply-address input; the routing log
`Reply phase ... -> mradxydzbsbdfoavedzx_at_..._yudehur@sl.local` [`email_handler.py`:L2196] confirms `is_reverse_alias`
sent it into `handle_reply`, and `is_reverse_alias(reply_email) -> True` was printed directly. For a *clean* key the
normalization is the identity (`normalize_reply_email(input)` returned the same string) — the transform only matters for
"strange" characters (see R6).

---

## R2 — Contact identification

**Direct answer:** The reply email is used in exactly one lookup: `contact = Contact.get_by(reply_email=reply_email)` [`email_handler.py`:L986]. `ModelMixin.get_by` is `return Session.query(cls).filter_by(**kw).first()` [`app/models.py`:L83-L84] — an **unordered `LIMIT 1`**. The emitted SQL has **no `ORDER BY`** (captured in the E3 setup below).

**Evidence (from E1, above):** the spy shows the resolving call and its result:

```
[SPY Contact.get_by] kwargs={'reply_email': 'mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local'} -> Contact(id=324, alias_id=1564, user_id=901, website_email='mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com')
```

The earlier spy line `kwargs={'reply_email': 'user_c9zn2wnjoq@mailbox.test'} -> None` is `handle()` pre-checking whether
the *sender* is itself a reverse-alias; it is unrelated to the reverse-alias resolution and returns `None` as expected.

The **exact emitted SQL** for this lookup is captured in the E3 setup (`str(query)` and `str(query.limit(1))`, `app/models.py`:L84). It ends in `... WHERE contact.reply_email = %(reply_email_1)s LIMIT %(param_1)s` with **no `ORDER BY`** — so with more than one matching row, PostgreSQL is free to return any of them. This is the mechanism behind R5/R8.

---

## R3 — Runtime values observed

**Direct answer:** For a normal reply, the concrete resolved values are:

| Value | Where captured (`file:line`) | Observed in E1 |
|---|---|---|
| extracted `reply_email` (envelope recipient, post-normalize) | `email_handler.py`:L972, L984 | `mradxydzbsbdfoavedzx_at_mradxydzbsbdfoavedzx_com_yudehur@sl.local` |
| resolved `Contact` | `email_handler.py`:L986 | `id=324, alias_id=1564, user_id=901` |
| `alias = contact.alias` | `email_handler.py`:L994 | `Alias 1564` (`swedes_smiles955@sl.local`) |
| `user = alias.user` | `email_handler.py`:L1004 | `User 901` (`user_c9zn2wnjoq@mailbox.test`) |
| authorized `mailbox` | `email_handler.py`:L1019 / L1364 | `Mailbox 1037` (`user_c9zn2wnjoq@mailbox.test`) |
| outbound relay target `contact.website_email` | `email_handler.py`:~L1226 | `mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com` |
| SMTP status | `app/email/status.py`:L2 | `250 Message accepted for delivery` (`E200`) |
| `EmailLog` | `email_handler.py`:L1042-L1050 | `id=595, is_reply=True, alias_id=1564, user_id=901, mailbox_id=1037` |

The handler prints all four resolved objects in one line at `email_handler.py`:L1051:

```
2026-07-06 23:15:41,918 - SL - DEBUG - 2969 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 595> for <Contact 324 mradxydzbsbdfoavedzx@mradxydzbsbdfoavedzx.com 1564>, <User 901 Test User user_c9zn2wnjoq@mailbox.test>, <Mailbox 1037 user_c9zn2wnjoq@mailbox.test>
```

and the outbound relay target is confirmed by `SendRequest.envelope_to == Contact1.website_email? True`. In the normal case, all of these are internally consistent (Contact → its own alias → that alias's user → that user's mailbox), so the reply is correctly relayed to the external sender behind the reverse-alias.

---

## R4 — Behavior across multiple reply events

**Direct answer:** When `reply_email` is unique (the normal case), resolution is **completely stable**: the same input resolves to the same Contact every time, both within one process and across separate process runs. There is **no** run-to-run inconsistency in the absence of a duplicate `reply_email`.

**Experiment E2.** Repeat the *exact* E1-style input `N=12` times in a loop, and repeat the whole thing in **two separate process runs** (two fresh interpreters, distinct PIDs). Report the distribution of the resolved `Contact.id`.

Command + complete unedited output — **run 1** (banner filtered):

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e2_stability.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
[E2] pid=2992  Seeded Contact1.id=325 alias_id=1566 user_id=902
[E2] reply_email (SAME unchanged input for all 12 events) = sqejvuhburicdwshsptf_at_sqejvuhburicdwshsptf_com_zgytkfp@sl.local
[E2] resolved Contact.id per event (n=12): [325, 325, 325, 325, 325, 325, 325, 325, 325, 325, 325, 325]
[E2] distribution of resolved Contact.id : {325: 12}
[E2] distribution of SMTP status         : {'250 Message accepted for delivery': 12}
[E2] all resolved to Contact1.id=325? True
[E2] DONE
```

Command + complete unedited output — **run 2** (separate process, banner filtered):

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e2_stability.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
[E2] pid=3005  Seeded Contact1.id=326 alias_id=1568 user_id=903
[E2] reply_email (SAME unchanged input for all 12 events) = leqpkvquhyetsmzjlckh_at_leqpkvquhyetsmzjlckh_com_npqsxifpj@sl.local
[E2] resolved Contact.id per event (n=12): [326, 326, 326, 326, 326, 326, 326, 326, 326, 326, 326, 326]
[E2] distribution of resolved Contact.id : {326: 12}
[E2] distribution of SMTP status         : {'250 Message accepted for delivery': 12}
[E2] all resolved to Contact1.id=326? True
[E2] DONE
```

**Scale & stability:** 12 events per run × 2 separate process runs = 24 observations. Within each run the distribution collapses to a single value (`{325: 12}`, then `{326: 12}`); every event returned `E200`. The id differs *between* runs only because each run seeds its own fresh Contact (325 vs 326), but is constant *within and across* the repeated events. Conclusion: with a unique `reply_email`, multi-event resolution is deterministic and correct (R4). This is the baseline the design intends, and it is exactly why the vulnerability in R5/R8 is *latent* rather than routinely observed.

---

## R5 — Same reply email → different Contacts over time

**Direct answer:** **Yes — once a duplicate `reply_email` exists, the same reply email can resolve to *different* Contacts** (and therefore different aliases/users). Resolution is an unordered `LIMIT 1` [`app/models.py`:L84], so PostgreSQL returns a database-arbitrary matching row; when the physical row order changes, the *same unchanged input* resolves to a *different* Contact. A duplicate is possible at all because the `reply_email` index is **non-unique** (`index=True`, no unique constraint) [`app/models.py`:L1899; migration `2021_071310_78403c7b8089_.py`:L22].

**Experiment E3 — setup (administratively/programmatically introduced duplicate).** I insert a second Contact under a *different* Alias/User with the **same** `reply_email` as the first. This is only possible because there is no DB uniqueness on `reply_email`; the generator (`generate_reply_email` + `available_sl_email`) would normally refuse it, so I bypass the generator by passing an explicit `reply_email` to `Contact.create` — and I **label this an admin/programmatic duplicate**, not something the generator produces on its own.

Command + complete unedited output (E3 setup — insert succeeds; SQL has no ORDER BY; banner filtered):

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e3_setup.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
[E3-setup] shared reply_email (the duplicate key) = uiqczjgwusvdqlvnbqfcgkbnwdqsoy@sl.local
[E3-setup] inserted Contact A id=327 alias_id=1570 user_id=904 website_email=tpkftpdtedpfdswezxoj@tpkftpdtedpfdswezxoj.com
[E3-setup] inserted Contact B id=328 alias_id=1572 user_id=905 website_email=xetzaofnjbutbaewofii@xetzaofnjbutbaewofii.com
[E3-setup] SECOND insert with duplicate reply_email SUCCEEDED (no uniqueness error).
[E3-setup] rows matching reply_email now (before/after enumeration) = 2:
    Contact(id=327, alias_id=1570, user_id=904, website_email='tpkftpdtedpfdswezxoj@tpkftpdtedpfdswezxoj.com')
    Contact(id=328, alias_id=1572, user_id=905, website_email='xetzaofnjbutbaewofii@xetzaofnjbutbaewofii.com')
[E3-setup] emitted SQL for Contact.get_by(reply_email=...) (models.py:L84 filter_by(...).first()):
    filter_by query : SELECT contact.id AS contact_id, contact.created_at AS contact_created_at, contact.updated_at AS contact_updated_at, contact.user_id AS contact_user_id, contact.alias_id AS contact_alias_id, contact.name AS contact_name, contact.website_email AS contact_website_email, contact.website_from AS contact_website_from, contact.reply_email AS contact_reply_email, contact.is_cc AS contact_is_cc, contact.pgp_public_key AS contact_pgp_public_key, contact.pgp_finger_print AS contact_pgp_finger_print, contact.mail_from AS contact_mail_from, contact.invalid_email AS contact_invalid_email, contact.block_forward AS contact_block_forward, contact.automatic_created AS contact_automatic_created, contact.flags AS contact_flags FROM contact WHERE contact.reply_email = %(reply_email_1)s
    .first() adds   : SELECT contact.id AS contact_id, contact.created_at AS contact_created_at, contact.updated_at AS contact_updated_at, contact.user_id AS contact_user_id, contact.alias_id AS contact_alias_id, contact.name AS contact_name, contact.website_email AS contact_website_email, contact.website_from AS contact_website_from, contact.reply_email AS contact_reply_email, contact.is_cc AS contact_is_cc, contact.pgp_public_key AS contact_pgp_public_key, contact.pgp_finger_print AS contact_pgp_finger_print, contact.mail_from AS contact_mail_from, contact.invalid_email AS contact_invalid_email, contact.block_forward AS contact_block_forward, contact.automatic_created AS contact_automatic_created, contact.flags AS contact_flags FROM contact WHERE contact.reply_email = %(reply_email_1)s LIMIT %(param_1)s
[E3-setup] persisted state: {'reply_email': 'uiqczjgwusvdqlvnbqfcgkbnwdqsoy@sl.local', 'contactA': 327, 'aliasA': 1570, 'userA': 904, 'mailboxA': 'user_h9bzv7bdee@mailbox.test', 'contactB': 328, 'aliasB': 1572, 'userB': 905, 'mailboxB': 'user_ivzm5d53ml@mailbox.test'}
[E3-setup] DONE
```

The `.first()` SQL ends in `... WHERE contact.reply_email = %(reply_email_1)s LIMIT %(param_1)s` — **no `ORDER BY`** — so with two matching rows PostgreSQL returns whichever the scan reaches first.

**Same-input resolution (E3 observe), run repeatedly across ≥2 process runs.** With the duplicate in place, drive the identical `handle()` reply and also call `Contact.get_by` directly, 10× each, in two separate processes:

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e3_observe.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
[E3-observe] pid=3041  SAME duplicate reply_email = uiqczjgwusvdqlvnbqfcgkbnwdqsoy@sl.local
[E3-observe] contactA=327 (alias 1570, user 904)  contactB=328 (alias 1572, user 905)
[E3-observe] rows matching reply_email = 2 -> ids [327, 328]
[E3-observe] SQL (no ORDER BY): ...ontact WHERE contact.reply_email = %(reply_email_1)s LIMIT %(param_1)s
[E3-observe] direct Contact.get_by(reply_email).first() ids (M=10): [327, 327, 327, 327, 327, 327, 327, 327, 327, 327]
[E3-observe]   distribution: {327: 10}
[E3-observe] handle() resolved Contact.id      (M=10): [327, 327, 327, 327, 327, 327, 327, 327, 327, 327]
[E3-observe]   distribution: {327: 10}
[E3-observe] handle() SMTP status distribution        : {'250 Message accepted for delivery': 10}
[E3-observe] DONE

$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e3_observe.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
[E3-observe] pid=3054  SAME duplicate reply_email = uiqczjgwusvdqlvnbqfcgkbnwdqsoy@sl.local
[E3-observe] contactA=327 (alias 1570, user 904)  contactB=328 (alias 1572, user 905)
[E3-observe] rows matching reply_email = 2 -> ids [327, 328]
[E3-observe] SQL (no ORDER BY): ...ontact WHERE contact.reply_email = %(reply_email_1)s LIMIT %(param_1)s
[E3-observe] direct Contact.get_by(reply_email).first() ids (M=10): [327, 327, 327, 327, 327, 327, 327, 327, 327, 327]
[E3-observe]   distribution: {327: 10}
[E3-observe] handle() resolved Contact.id      (M=10): [327, 327, 327, 327, 327, 327, 327, 327, 327, 327]
[E3-observe]   distribution: {327: 10}
[E3-observe] handle() SMTP status distribution        : {'250 Message accepted for delivery': 10}
[E3-observe] DONE
```

For a *static* pair of duplicate rows the selection happened to be stable (Contact 327, the lower `ctid`, in both runs). Stability of the SAME input across ≥2 runs is thus confirmed — **but the selected row is chosen by physical order, not by any deterministic rule the design guarantees.** To prove the "different Contacts over time" claim decisively, I change the physical order and re-run the SAME lookup.

**Experiment E3 — flip (decisive proof of R5): SAME input, different result over time.** A non-HOT `UPDATE` to Contact 327 (touching an unrelated column, `name`) rewrites its heap tuple, moving it after Contact 328 in scan order. Observe the SAME `get_by(reply_email=...)` before and after:

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e3_flip.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
[E3-flip] duplicate reply_email = uiqczjgwusvdqlvnbqfcgkbnwdqsoy@sl.local
[E3-flip] BEFORE  heap order (id, ctid, user_id, alias_id): [(327, '(2,31)', 904, 1570), (328, '(2,32)', 905, 1572)]
[E3-flip] BEFORE  Contact.get_by(reply_email).first() -> Contact id=327 user_id=904 alias_id=1570
[E3-flip] DURING  unrelated UPDATE: touch Contact id=327.name (commit)
[E3-flip] AFTER   heap order (id, ctid, user_id, alias_id): [(328, '(2,32)', 905, 1572), (327, '(2,33)', 904, 1570)]
[E3-flip] AFTER   Contact.get_by(reply_email).first() -> Contact id=328 user_id=905 alias_id=1572
[E3-flip] SAME input resolved to a DIFFERENT Contact over time? True  (before=327 -> after=328)
[E3-flip] DONE
```

**Reasoning:** `filter_by(reply_email=...).first()` emits `... WHERE reply_email = ? LIMIT 1` with **no `ORDER BY`**
(shown in E3 setup), so PostgreSQL returns whichever matching tuple the scan reaches first. That physical order is not
stable over time — an ordinary `UPDATE` (a routine event: the app updates `Contact.name`, flags, PGP fields, etc.)
rewrites a row's `ctid` (here `327` moved from `(2,31)` to `(2,33)`) and reorders equal-key rows. The captured
before/after shows the identical unchanged input `uiqczjgwusvdqlvnbqfcgkbnwdqsoy@sl.local` resolving first to Contact 327
(user 904) and later to Contact 328 (user 905). **Direct answer confirmed: the same reply email CAN resolve to different
Contacts (and users) over time once a duplicate exists.**


---

## R6 — Temporary non-resolution

**Direct answer:** **Yes — resolution can fail.** When `Contact.get_by(reply_email=...)` returns `None`, `handle_reply` logs `No contact with ... as reverse alias` [`email_handler.py`:L988] and returns `status.E502` = `"550 SL E502 Email not exist"` [`app/email/status.py`:L39]. This is **temporary** in the ordinary sense (the same address resolves as soon as the matching Contact exists), but it can be effectively **permanent** for a stored Contact whose `reply_email` contains a character that `normalize_reply_email` rewrites — a **normalization asymmetry** that makes the Contact unreachable via its own address. A wrong reply *domain* fails earlier with `E501` [`email_handler.py`:L977-L981].

**Experiment E4.** Command + complete unedited output (banner filtered):

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e4_nonresolution.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
2026-07-06 23:17:29,021 - SL - INFO - 3100 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-06 23:17:29,034 - SL - DEBUG - 3100 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email cackle_mostly866@sl.local
2026-07-06 23:17:29,041 - SL - INFO - 3100 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
==============================================================================
E4(a) - valid reverse-alias, NO matching Contact -> E502
==============================================================================
[E4a] rcpt_to = reply+yauebblmzdwqgcmyghvj@sl.local
[E4a] is_reverse_alias(rcpt_to) = True  (routes to handle_reply via reply+ prefix)
[E4a] BEFORE: Contact rows with this reply_email = 0
2026-07-06 23:17:29,050 - SL - DEBUG - 3100 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-06 23:17:29,051 - SL - DEBUG - 3100 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_xqtbmzpbav@mailbox.test, rcpt_tos:['reply+yauebblmzdwqgcmyghvj@sl.local'], header_from:None, header_to:reply+yauebblmzdwqgcmyghvj@sl.local, cc:None, reply-to:None, message_id:<e4@x.local>, client_ip:None, headers:[('To', 'reply+yauebblmzdwqgcmyghvj@sl.local'), ('Subject', 'E4'), ('Message-ID', '<e4@x.local>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-06 23:17:29,053 - SL - DEBUG - 3100 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_xqtbmzpbav@mailbox.test(None) -> reply+yauebblmzdwqgcmyghvj@sl.local
2026-07-06 23:17:29,054 - SL - WARNING - 3100 - "/app/email_handler.py:988" - handle_reply() -  - No contact with reply+yauebblmzdwqgcmyghvj@sl.local as reverse alias
[E4a] AFTER : Contact rows with this reply_email = 0 (stays absent)
[E4a] handle() status = '550 SL E502 Email not exist' ; status.E502 = '550 SL E502 Email not exist' ; match=True

==============================================================================
E4(b) - normalization asymmetry: raw matches routing, normalized key misses -> E502
==============================================================================
[E4b] stored Contact id=329 reply_email='reply+gsccbxbpqu#x@sl.local' (legacy value with disallowed '#')
[E4b] is_reverse_alias(raw) = True  (raw address MATCHES an existing Contact -> True -> handle_reply)
[E4b] normalize_reply_email(raw) = 'reply+gsccbxbpqu_x@sl.local'  (direct-call illustration; email_validation.py:L25-38)
[E4b] get_by(raw)        -> Contact 329 (the stored row)
[E4b] get_by(normalized) -> None (what handle_reply actually looks up at L986)
2026-07-06 23:17:29,069 - SL - DEBUG - 3100 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-06 23:17:29,070 - SL - DEBUG - 3100 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_xqtbmzpbav@mailbox.test, rcpt_tos:['reply+gsccbxbpqu#x@sl.local'], header_from:None, header_to:reply+gsccbxbpqu#x@sl.local, cc:None, reply-to:None, message_id:<e4@x.local>, client_ip:None, headers:[('To', 'reply+gsccbxbpqu#x@sl.local'), ('Subject', 'E4'), ('Message-ID', '<e4@x.local>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-06 23:17:29,072 - SL - DEBUG - 3100 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_xqtbmzpbav@mailbox.test(None) -> reply+gsccbxbpqu#x@sl.local
2026-07-06 23:17:29,073 - SL - WARNING - 3100 - "/app/email_handler.py:988" - handle_reply() -  - No contact with reply+gsccbxbpqu_x@sl.local as reverse alias
[E4b] handle() status = '550 SL E502 Email not exist' ; status.E502 = '550 SL E502 Email not exist' ; match=True
[E4b] => the stored Contact is UNREACHABLE via its own reverse-alias (permanent until the stored value is corrected)

==============================================================================
E4(c) - wrong reply domain -> E501
==============================================================================
[E4c] stored Contact id=330 reply_email='reply+eiutprshkuam@foreign.example'
[E4c] is_reverse_alias(rcpt_to) = True  (matches Contact -> handle_reply)
2026-07-06 23:17:29,080 - SL - DEBUG - 3100 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-06 23:17:29,081 - SL - DEBUG - 3100 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_xqtbmzpbav@mailbox.test, rcpt_tos:['reply+eiutprshkuam@foreign.example'], header_from:None, header_to:reply+eiutprshkuam@foreign.example, cc:None, reply-to:None, message_id:<e4@x.local>, client_ip:None, headers:[('To', 'reply+eiutprshkuam@foreign.example'), ('Subject', 'E4'), ('Message-ID', '<e4@x.local>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-06 23:17:29,083 - SL - DEBUG - 3100 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_xqtbmzpbav@mailbox.test(None) -> reply+eiutprshkuam@foreign.example
2026-07-06 23:17:29,084 - SL - WARNING - 3100 - "/app/email_handler.py:980" - handle_reply() -  - Reply email reply+eiutprshkuam@foreign.example has wrong domain
[E4c] handle() status = '550 SL E501' ; status.E501 = '550 SL E501' ; match=True
[E4] DONE
```

**Reasoning & before/during/after:** In E4a the Contact is **absent before** (`0` rows), the handler emits the L988 warning **during**, and the row is **still absent after** — so this non-resolution is *temporary* by nature (create the Contact and the same address resolves). E4b is the subtle case: the stored `reply_email` literally contains `#`, which `sanitize_email` (`app/utils.py`:L97) preserves, so `is_reverse_alias` on the raw address is `True` and the row is findable by its literal value — **but** the handler normalizes the inbound key at L984 (`#`→`_`, because `#` is not in `_ALLOWED_CHARS` [`app/email_validation.py`:L9]) *before* the L986 lookup, so `get_by` misses (`No contact with reply+gsccbxbpqu_x@sl.local`) and returns `E502`. The Contact is effectively **permanently** unreachable via its own address until the stored value is corrected. E4c fails even earlier at the domain guard (`E501`, L980).

**Supplementary (direct-call illustration): the normalization rewrite across several characters.** `normalize_reply_email` [`app/email_validation.py`:L25-L38] vs `sanitize_email` [`app/utils.py`:L97]:

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/probe_norm.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
input='reply+ab#cd@sl.local'
  sanitize_email -> 'reply+ab#cd@sl.local'
  normalize_reply_email -> 'reply+ab_cd@sl.local'
  changed_by_normalize=True
input='reply+ab cd@sl.local'
  sanitize_email -> 'reply+abcd@sl.local'
  normalize_reply_email -> 'reply+abcd@sl.local'
  changed_by_normalize=False
input='ra+xy%zt@sl.local'
  sanitize_email -> 'ra+xy%zt@sl.local'
  normalize_reply_email -> 'ra+xy_zt@sl.local'
  changed_by_normalize=True
input='reply+aébc@sl.local'
  sanitize_email -> 'reply+aébc@sl.local'
  normalize_reply_email -> 'reply+aebc@sl.local'
  changed_by_normalize=True
```

Characters disallowed by `_ALLOWED_CHARS` (`#`, `%`) are rewritten to `_`, and non-ASCII (`é`) is transliterated (`convert_to_id`, `app/email_validation.py`:L27-L28) — any of which shifts the lookup key away from a stored value that contained the original character.

---

## R7 — Correct-now-wrong-later (anti-spoofing outcome)

**Direct answer:** A mis-resolved reply (pointing at the *wrong* alias/user) is **usually rejected**, not silently delivered: `get_mailbox_from_mail_from(mail_from, alias)` [`email_handler.py`:L1019, def L1364] returns `None` when the replier's `mail_from` is not a mailbox/authorized-address of the *resolved* alias, and the handler then returns `status.E214` = `"250 SL E214 Unauthorized for using reverse alias"` [`app/email/status.py`:L22] (a `250` to avoid backscatter). **However**, a mis-resolution *does* become a **silent delivery to the wrong user** in two concrete situations: (i) the wrongly-resolved alias has `disable_email_spoofing_check = True`, so the handler falls back to `mailbox = alias.mailbox` [`email_handler.py`:L1021-L1029]; or (ii) the replier's `mail_from` is *also* a valid mailbox of the wrongly-resolved alias (e.g., two aliases share a mailbox). So the anti-spoofing gate is a **mitigating control**, not a guarantee.

**Experiment E5.** Using a cross-user duplicate `reply_email` (Contact B inserted first → `get_by` resolves to B, whose owner is *not* the replier), drive the identical reply under three configurations.

Command + complete unedited output (banner filtered):

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e5_antispoof.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
==============================================================================
E5 SCENARIO X - CROSS-USER mis-resolution via duplicate reply_email
==============================================================================
[E5X] duplicate reply_email R = ubvzylwnxkolcnvcritzeqjkvmeuhc@sl.local
[E5X] Contact A id=332(user 907, alias 1576)  Contact B id=331(user 908, alias 1578)
[E5X] get_by(R) RESOLVES TO Contact id=331 -> alias 1578, user 908
[E5X] the actual REPLIER mailbox = user_evze68a1l3@mailbox.test (belongs to user 907, NOT the resolved contact's user 908)

--- (i) DEFAULT (disable_email_spoofing_check=False): expect E214 bounce ---
2026-07-06 23:17:41,851 - SL - WARNING - 3121 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: user_evze68a1l3@mailbox.test. msg from header: None, reverse-alias ubvzylwnxkolcnvcritzeqjkvmeuhc@sl.local, <Alias 1578 dachas_worker050@sl.local> <User 908 Test User user_dctm5mmgul@mailbox.test> <Contact 331 fcfomrfgkorrfiftbgcv@fcfomrfgkorrfiftbgcv.com 1578>
[E5X-i] handle() status = '250 SL E214 Unauthorized for using reverse alias' ; status.E214 = '250 SL E214 Unauthorized for using reverse alias' ; match=True
[E5X-i] outbound messages = ['user_dctm5mmgul@mailbox.test']  (none => NOT delivered to the wrong user)

--- (ii) resolved alias.disable_email_spoofing_check=True: expect SILENT delivery to WRONG user ---
2026-07-06 23:17:41,895 - SL - WARNING - 3121 - "/app/email_handler.py:1023" - handle_reply() -  - ignore unknown sender to reverse-alias user_evze68a1l3@mailbox.test: <Alias 1578 dachas_worker050@sl.local> -> <Contact 331 fcfomrfgkorrfiftbgcv@fcfomrfgkorrfiftbgcv.com 1578>
2026-07-06 23:17:41,907 - SL - WARNING - 3121 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
[E5X-ii] handle() status = '250 Message accepted for delivery' ; status.E200 = '250 Message accepted for delivery' ; match=True
[E5X-ii] outbound envelope_to = ['fcfomrfgkorrfiftbgcv@fcfomrfgkorrfiftbgcv.com'] (== resolved Contact.website_email 'fcfomrfgkorrfiftbgcv@fcfomrfgkorrfiftbgcv.com'? True)
[E5X-ii] EmailLog for resolved contact: before=0 after=1
[E5X-ii] EmailLog(id=640, is_reply=True, user_id=908, alias_id=1578, mailbox_id=1044)
[E5X-ii] => message from replier(user 907) was relayed on behalf of WRONG user 908 to 'fcfomrfgkorrfiftbgcv@fcfomrfgkorrfiftbgcv.com'

==============================================================================
E5 SCENARIO Y - SAME-USER two aliases SHARING a mailbox (mail_from matches)
==============================================================================
[E5Y] user 909; alias1=1580 mailbox=user_q0s7r9krom@mailbox.test; alias2=1581 mailbox=user_q0s7r9krom@mailbox.test (same mailbox=True)
[E5Y] duplicate reply_email R2=ibydhqkegqpbrmgokbprjvzeevphvy@sl.local; contact1 id=333(alias 1580) website=intended-A@ext.example; contact2 id=334(alias 1581) website=intended-B@ext.example
[E5Y] get_by(R2) RESOLVES TO contact id=333 (alias 1580) website_email='intended-A@ext.example'
2026-07-06 23:17:42,238 - SL - WARNING - 3121 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
[E5Y] replier = user's own mailbox user_q0s7r9krom@mailbox.test -> anti-spoofing PASSES (shared mailbox)
[E5Y] handle() status = '250 Message accepted for delivery' ; match E200=True
[E5Y] outbound envelope_to = ['intended-A@ext.example'] -> relayed to the ARBITRARILY-selected contact 'intended-A@ext.example'
[E5] DONE
```

**Reasoning & sub-cases:**
- **(i) default (`disable_email_spoofing_check=False`):** the replier `user_evze68a1l3@mailbox.test` (user 907) is not a mailbox of the resolved Alias 1578 (user 908), so `get_mailbox_from_mail_from` returns `None`, `handle_unknown_mailbox` fires with the L1393 warning (`Reply email can only be used by mailbox. Actual mail_from: ...`), and the reply is rejected with `E214`. **The mis-resolved reply is NOT relayed to the external sender** — the only outbound is the unknown-reverse-alias alert to the resolved alias's own mailbox (`user_dctm5mmgul@mailbox.test`). This is the common, protective outcome.
- **(ii) `disable_email_spoofing_check=True` on the resolved alias:** the handler logs `ignore unknown sender to reverse-alias ...` (L1023), falls back to `mailbox = alias.mailbox`, and **delivers**. `EmailLog id=640` shows the reply relayed under `user_id=908` (the wrong user), to Contact B's `website_email`. This is a **silent wrong-user delivery** — the crux of R7: the replier is user 907 but the message is relayed on behalf of user 908.
- **(Y) shared mailbox:** when the replier's `mail_from` is a valid mailbox of the resolved alias (here one user owns both aliases and the mailbox), the gate passes and the reply is delivered to the arbitrarily-selected Contact (`intended-A@ext.example`) — the wrong contact relative to the user's intent, though within the same user.

So "correct now, wrong later" is real: the *same* reply that resolves correctly today can, after a duplicate is introduced and/or physical order changes (R5), resolve to a different alias/user; whether that becomes a bounce (`E214`) or a **silent misdelivery** depends entirely on the anti-spoofing configuration of the wrongly-resolved alias.


---

## R8 — Root causes (race conditions / uniqueness assumptions / timing)

**Direct answer:** The reply-by-`reply_email` lookup rests on **four** interlocking facts, three of which are latent weaknesses:

1. **Uniqueness assumption (not DB-enforced).** Reply routing assumes `reply_email` identifies exactly one Contact, but the *only* DB unique constraint on `contact` is `uq_contact(alias_id, website_email)` [`app/models.py`:L1874-L1876; migration `2020_031711_0809266d08ca_.py`:L45]. `reply_email` is merely `index=True` — **non-unique** [`app/models.py`:L1899; migration `2021_071310_78403c7b8089_.py`:L22]. Nothing at the database level prevents two Contacts from sharing a `reply_email` (proven in E3: the second insert succeeded).
2. **Soft check-then-act (TOCTOU) uniqueness.** Uniqueness is enforced only at *generation* time: `generate_reply_email` [`app/email_utils.py`:L1103] loops (`for _ in range(1000):` L1136) and returns a candidate only `if available_sl_email(reply_email)` [L1150-L1151]. `available_sl_email` [`app/models.py`:L1425-L1432] returns `False` if any `Alias`, `Contact.reply_email`, or `DeletedAlias` already uses the address — a **check-then-act** read with no lock and no transaction spanning the later insert, so two concurrent generations could both observe "available" and both insert (no constraint would stop them).
3. **Unordered selection.** The lookup is `filter_by(reply_email=...).first()` = `... LIMIT 1` with **no `ORDER BY`** [`app/models.py`:L84] (SQL captured in R2/R5), so with a duplicate present the chosen row is database-arbitrary and can change over time (E3 flip).
4. **Contact-creation race is guarded — but only for `(alias, website_email)`, not `reply_email`.** `create_contact` [`app/contact_utils.py`:L42] relies on `uq_contact` + `IntegrityError` rollback/refetch [L113-L119] to converge concurrent creations for the same `(alias, sender)`. That guard does **not** extend to `reply_email`.

**Experiment E6 — the contact-creation race and what it does / does not protect.** Command + complete unedited output (banner filtered):

```
$ docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/python /tmp/sl_investigation/e6_integrity_race.py' \
    | grep -v -E "load config file|>>> URL:|WARNING: Use a temp|Upload files to local|>>> init logging|load words file"
2026-07-06 23:18:01,582 - SL - INFO - 3142 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-06 23:18:01,601 - SL - INFO - 3142 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
[E6] alias_id=1583 website_email=fvncqvosdlhjltmijhcr@fvncqvosdlhjltmijhcr.com
[E6] BEFORE: Contact rows for (alias_id=1583, website_email) = 0
[E6] DURING: first Contact.create OK -> id=335 reply_email=jlttsqkfkznuhjhffngsgywjxytmtp@sl.local
[E6] DURING: second Contact.create for SAME (alias_id, website_email) with a NEW reply_email=myygighlwflbtdufycoxxekczwwcsm@sl.local
[E6] DURING: caught IntegrityError (this is what create_contact catches at contact_utils.py:L113):
      duplicate key value violates unique constraint "uq_contact" DETAIL: Key (alias_id, website_email)=(1583, fvncqvosdlhjltmijhcr@fvncqvosdlhjltmijhcr.com) already exists.
[E6] DURING: after rollback+refetch (L114-L118) -> Contact id=335 reply_email=jlttsqkfkznuhjhffngsgywjxytmtp@sl.local
[E6] AFTER : Contact rows for (alias_id, website_email) = 1 (converged to ONE)
[E6] AFTER : the loser's generated reply_email R2 exists anywhere? False (discarded on rollback => this path creates NO duplicate reply_email)
[E6] AFTER : surviving reply_email == R1? True

[E6] Higher-level convergence via the REAL create_contact() (automatic_created):
[E6] create_contact x2 for same (alias, xbvxzuegudkcheqisyiz@xbvxzuegudkcheqisyiz.com) -> ids (337, 337) same=True; created flags=(True,False)
[E6] both share reply_email=xbvxzuegudkcheqisyiz_at_xbvxzuegudkcheqisyiz_com_bzeygtvuqi@sl.local (single generated value; uq_contact made the 2nd call idempotent)
[E6] CONTRAST: uq_contact guards (alias_id, website_email); NO unique constraint guards reply_email [app/models.py:L1874-L1876 vs L1899; migration ...78403c7b8089_.py:L22].
[E6] DONE
```

**Reasoning (before / during / after):** **before**, no Contact exists for `(alias 1583, fvncqvosdlhjltmijhcr@…)`; **during**, the first `Contact.create` succeeds (id 335) and the second — same `(alias_id, website_email)`, but a *freshly generated* `reply_email` R2 — hits `uq_contact` and raises the verbatim `duplicate key value violates unique constraint "uq_contact" DETAIL: Key (alias_id, website_email)=(1583, fvncqvosdlhjltmijhcr@fvncqvosdlhjltmijhcr.com) already exists.`; `create_contact` catches it, rolls back, and refetches; **after**, exactly one Contact (id 335) survives and both callers converge on it. Crucially, the loser's generated `reply_email` **R2 is discarded on rollback** (`exists anywhere? False`), so the *contact-creation* path itself never leaves a duplicate `reply_email`. The higher-level `create_contact()` is likewise idempotent (`ids (337, 337)`, `created flags=(True,False)`). The point of R8 is the **asymmetry**: `(alias, website_email)` is protected by a real constraint and a convergence mechanism, while `reply_email` has neither — its uniqueness rests entirely on the soft TOCTOU check (#2) and, once violated, on an unordered `first()` (#3).

**Calibrated risk (this is the key nuance).** A *natural* collision is astronomically unlikely: the no-sender branch draws a random local part of `random.randint(20, 50)` characters [`app/email_utils.py`:L1145] from a large alphabet, so `available_sl_email` essentially never sees a real clash, and E2 confirms normal multi-event resolution is perfectly stable (R4). Therefore the practical exposure is **latent**: it requires a duplicate `reply_email` that is *administratively or programmatically* introduced (bulk import, manual DB edit, a future code path that sets `reply_email` explicitly, or a genuine concurrent double-generate under the TOCTOU window), after which the unordered `first()` (E3) selects a database-arbitrary Contact — and that selection can even change over time (E3 flip). It is a latent uniqueness/timing weakness, not a routinely-fired defect.

---

## R9 — Cause → Effect

**Direct answer:** The reply-resolution chain is:

> inbound reply address = **SMTP envelope recipient** `reply_email = rcpt_to` [`email_handler.py`:L972]
> → **normalized** `normalize_reply_email(reply_email)` [`email_handler.py`:L984; asymmetry at `app/email_validation.py`:L25-L38]
> → **Contact lookup** `Contact.get_by(reply_email=...)` [`email_handler.py`:L986] = `filter_by(...).first()` = **unordered `LIMIT 1`** [`app/models.py`:L84]
> → **destination selected** `alias = contact.alias` [L994], `user = alias.user` [L1004], `mailbox = get_mailbox_from_mail_from(mail_from, alias)` [L1019/L1364]
> → **relay** to `contact.website_email` [~L1226].

**How the observed values lead to misrouting (cause → effect):**

1. **Cause:** `reply_email` has no DB-level uniqueness (R8 #1; `app/models.py`:L1899, migration L22) and is only softly de-duplicated at generation (R8 #2). **Effect:** a duplicate `reply_email` *can* exist across two different Contacts/aliases/users (E3 setup: second insert succeeded — Contact 327 and 328 share `uiqczjgwusvdqlvnbqfcgkbnwdqsoy@sl.local`).
2. **Cause:** the lookup is an unordered `LIMIT 1` (R2/R8 #3; SQL shows no `ORDER BY`). **Effect:** with a duplicate present, `get_by(...).first()` returns a **database-arbitrary** Contact, and that choice can flip when physical row order changes (E3 flip: same input → Contact 327/user 904, then Contact 328/user 905).
3. **Cause:** the handler then trusts that Contact's `alias`/`user` (L994/L1004) as the forwarding destination. **Effect:** a wrong Contact selects a **wrong alias → wrong user**, and the outbound is aimed at that Contact's `website_email` (E5: resolved to Contact 331 under user 908, not the replier's user 907).
4. **Mitigating control:** the anti-spoofing gate `get_mailbox_from_mail_from` (L1019/L1364) checks whether the replier's `mail_from` is a mailbox/authorized-address of the *resolved* alias. **Effect:** usually the mismatch yields `E214` and the reply is **not** relayed to the external sender (E5 case i) — so most mis-resolutions bounce rather than leak. **But** if the resolved alias has `disable_email_spoofing_check=True` (E5 case ii → `E200`, `EmailLog user_id=908`) or shares a mailbox with the replier (E5 case Y → `E200`), the gate passes and the reply is **silently delivered on behalf of the wrong user**.
5. **Contrast that bounds the risk:** the analogous *contact-creation* race is safely converged by `uq_contact` + `IntegrityError` rollback/refetch (E6), and natural `reply_email` collisions are astronomically unlikely (R8 calibrated risk). So the misrouting is a **latent** consequence of the uniqueness assumption + unordered selection, realized only once a duplicate `reply_email` is introduced.

**Net:** reply routing is correct and stable in normal operation (R4), because `reply_email` is effectively unique in practice. The defect is latent: **if** the softly-enforced uniqueness is ever violated, the unordered `get_by(...).first()` makes resolution depend on database-arbitrary, time-varying row order, which can select the wrong alias/user; whether that becomes an `E214` bounce or a silent wrong-user delivery is decided by the anti-spoofing configuration of the wrongly-resolved alias.

---

## Read-only guarantee (repository left unchanged)

All observation scripts lived in the container's own `/tmp/sl_investigation/` (which is **not** bind-mounted into the repository) and were removed afterward; no tracked file was edited. The working tree at the repository root contains **only** the new deliverable, and **no tracked file is modified** (`git diff --stat HEAD` is empty):

```
$ cd /tmp/blitzy/app/blitzy-c2a41c33-7e7e-4397-9fc6-7fc76ebf8165_9eedb3
$ git rev-parse --abbrev-ref HEAD && git rev-parse --short HEAD
blitzy-c2a41c33-7e7e-4397-9fc6-7fc76ebf8165
2cd6ee77

$ git diff --stat HEAD
$ echo "[empty output above = no tracked file modified]"
[empty output above = no tracked file modified]

$ git status --porcelain
?? blitzy/

$ find blitzy/ -type f
blitzy/documentation/app_2cd6ee777f8c.md
```

The single untracked path `blitzy/` holds exactly one file — this deliverable. No source, test, migration, or configuration file was added, modified, or deleted. (`static/upload/` appears only as a git-*ignored* pre-existing directory and is not part of this change.)

---

## Coverage pass R1–R9

- [x] **R1 — Reply-address derivation.** Reply address = SMTP envelope recipient `reply_email = rcpt_to` [`email_handler.py`:L972], normalized [L984]; routed to `handle_reply` via `is_reverse_alias(rcpt_to)` [L2195 → `app/email_utils.py`:L1156]. Evidence: **E1** (`is_reverse_alias -> True`, routing log L2196).
- [x] **R2 — Contact identification.** `Contact.get_by(reply_email=...)` [`email_handler.py`:L986] = `filter_by(...).first()` unordered `LIMIT 1` [`app/models.py`:L83-L84]. Evidence: **E1** spy line + **E3 setup** emitted SQL (no `ORDER BY`).
- [x] **R3 — Runtime values observed.** Extracted `reply_email`, resolved `Contact(id=324, alias_id=1564, user_id=901)`, `alias`=1564, `user`=901, `mailbox`=1037, outbound `website_email`, status `E200`, `EmailLog id=595 is_reply=True`. Evidence: **E1** (L1051 line + result table).
- [x] **R4 — Behavior across multiple reply events.** Stable: `{325:12}` then `{326:12}` across 2 process runs; all `E200`. Evidence: **E2** (SAME input, N=12, ≥2 runs).
- [x] **R5 — Same reply email → different Contacts over time.** **Yes** once a duplicate exists: unordered `first()` returns a database-arbitrary row; SAME input flipped `327(user 904) → 328(user 905)` after an unrelated `UPDATE` reordered the heap. Evidence: **E3 setup + E3 observe (≥2 runs) + E3 flip**.
- [x] **R6 — Temporary non-resolution.** `E502` "no contact" [L988] (temporary — resolves once Contact exists); normalization asymmetry (`#`→`_` at L984) makes a stored Contact permanently unreachable via its own address; `E501` wrong domain [L980]. Evidence: **E4a/E4b/E4c** + `probe_norm`.
- [x] **R7 — Correct-now-wrong-later.** Mis-resolution usually **bounces** with `E214` (gate `get_mailbox_from_mail_from` → `None`, L1019/L1364); **silently delivers to the wrong user** when `disable_email_spoofing_check=True` (L1021-L1029) or a shared mailbox lets `mail_from` match. Evidence: **E5 (i)/(ii)/(Y)** — silent wrong-user delivery captured as `EmailLog id=640 user_id=908` while the replier is user 907.
- [x] **R8 — Root causes.** (i) uniqueness assumption not DB-enforced (`uq_contact` covers only `(alias_id, website_email)` [`app/models.py`:L1874-L1876; migration L45]; `reply_email` non-unique [L1899; migration L22]); (ii) soft TOCTOU generation (`generate_reply_email` + `available_sl_email` [`app/email_utils.py`:L1136-L1151; `app/models.py`:L1425-L1432]); (iii) unordered `first()` [L84]; (iv) contact-creation race guarded only for `(alias, website_email)` via `uq_contact` + `IntegrityError` rollback/refetch [`app/contact_utils.py`:L113-L119], `reply_email` unguarded. Calibrated: **latent** (natural 20-50-char collision astronomically unlikely). Evidence: **E3 + E6**.
- [x] **R9 — Cause → Effect.** envelope recipient (L972) → normalized (L984) → unordered `get_by(...).first()` (L986 → L84) → `alias`/`user`/`mailbox` (L994/L1004/L1019) → relay to `website_email` (~L1226); a duplicate + arbitrary/time-varying selection picks the wrong alias/user; anti-spoofing (L1364) usually converts that to an `E214` bounce, except under `disable_email_spoofing_check`/shared-mailbox. Evidence: **E1–E6 synthesized**.

**All nine items answered by name and backed by executed commands with complete, unedited output and `file:line` citations.**

