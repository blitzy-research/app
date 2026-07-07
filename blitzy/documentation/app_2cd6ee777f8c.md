# Alias Email-Forwarding: Runtime Investigation of Log Text, Message-ID, `From` Header, and DB Records

## Summary

This document answers four questions about how SimpleLogin transforms an inbound email as it is forwarded through an alias. **Every value below was captured by running the real code** — driving the canonical SMTP routing entry point `email_handler.handle(envelope, msg)` (`email_handler.py:L1945`) inside the pinned Docker image, on the **canonical PostgreSQL 13** stack — and never inferred from reading code alone. Each observation was run at least twice (`run1`/`run2`) and the **complete, unedited output of both runs is pasted below**.

Headline findings:

- **Q1 (logs/status).** A successful forward emits a sequence of `LOG.d(...)` DEBUG lines from `forward_email_to_mailbox()` and returns SMTP status **`250 Message accepted for delivery`** (`E200`). A forward to a non-existent alias emits the `not exist` / `cannot be created on-the-fly, return 550` DEBUG lines from `handle_forward()` and returns **`550 SL E515 Email not exist`** (`E515`).
- **Q2 (SL Message-ID) — the central finding.** On a **forward**, SimpleLogin does **not** generate an "SL Message-ID." The original inbound `Message-ID` is **preserved byte-for-byte** on the outgoing message, and `EmailLog.sl_message_id` stays **`NULL`**. An SL Message-ID (`<{timeval}.{pid}.{randint}.{email_log.id}@sl.local>`) is minted **only in the reply phase**. This forward-vs-reply asymmetry is the most likely explanation for the reported "inconsistent behavior."
- **Q3 (`From` header).** The forwarded `From` is rewritten to an `"<local> at <domain>"` display name paired with the reverse-alias — e.g. `"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local>` in `run1`. The observed reverse-alias uses the **sender-included** format `{sanitized-sender}_{random}@sl.local` (not a pure-random local-part), because the canonical ORM default for `User.include_sender_in_reverse_alias` is `True`.
- **Q4 (DB records).** A single forward from a **new** sender creates exactly **three** rows: one `Contact`, one `UserAuditLog` (`action=create_contact`), and one `EmailLog`. `EmailLog.sl_message_id` is `NULL` and **no** `MessageIDMatching` row is created on a forward.

> **Diagnostic-only scope.** This is a read-only investigation. No source file was modified and no fix was applied, even though the Q2 forward-vs-reply distinction is the probable root cause of the reported inconsistency.

---

## Environment & method

### Canonical build / run recipe

The investigation ran inside the mandated pre-built image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (running container `sl-setup`), on branch source commit **`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`**. The canonical build/run recipe (confirmed from `Dockerfile`, `.github/workflows/main.yml`, `pyproject.toml`, and `tests/test.env`) is:

```bash
# Interpreter & OS packages (Dockerfile: `FROM python:3.10`; CI python-version '3.10'; pyproject `python = "^3.10"`)
sudo apt install -y libre2-dev libpq-dev          # .github/workflows/main.yml:L24,L84

# Python dependencies (Poetry-managed)
poetry install --no-interaction                   # .github/workflows/main.yml:L28,L88

# Provision schema via Alembic migrations (NOT create_all), against PostgreSQL 13
CONFIG=tests/test.env poetry run alembic upgrade head   # .github/workflows/main.yml:L98
```

Config comes from `tests/test.env`: `EMAIL_DOMAIN=sl.local` (`tests/test.env:L8`), `NOT_SEND_EMAIL=true` (`tests/test.env:L7`), `OTHER_ALIAS_DOMAINS` includes `sl.local` (`tests/test.env:L9`), `ALIAS_AUTOMATIC_DISABLE=true` (`tests/test.env:L62`), and `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:L17`).

### Canonical runtime actually used

To match the canonical CI configuration — which pins **`postgres:13`** (`.github/workflows/main.yml:L47`) published on host port **`15432`** (`.github/workflows/main.yml:L60`, `- 15432:5432`) and referenced by `tests/test.env:L17` — a **PostgreSQL 13** cluster was stood up on port `15432` and the schema provisioned with `alembic upgrade head`. All values reported below were produced against this canonical PostgreSQL 13 stack. The interpreter and libraries observed at runtime:


```
postgres: PostgreSQL 13.23 (Debian 13.23-1.pgdg12+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
python: 3.10.18 | sqlalchemy: 1.3.24 | flask: 1.1.2 | aiosmtpd: 1.4.2 | arrow: 0.16.0
```

> The `alembic upgrade head` provisioning must start from a **bare** `public` schema: migration `424808e1fe49` (`migrations/versions/2021_082012_424808e1fe49_.py`) runs `CREATE EXTENSION pg_trgm` and, if the extension already exists, issues a transactional `Rollback`; because Alembic runs the whole upgrade as one transactional-DDL transaction, pre-creating `pg_trgm` would roll back the already-created tables and the subsequent `CREATE INDEX ... ON alias` would fail. Letting the migration create the extension (as CI does) provisions all 77 tables cleanly.

### How observations were produced (real entry point)

All observations flow through the real router `email_handler.handle(envelope, msg)` (`email_handler.py:L1945`) — the same function `MailHandler.handle_DATA` (`email_handler.py:L2289`) / `MailHandler._handle` invoke for live SMTP. I did **not** call `handle_forward`, `forward_email_to_mailbox`, or `handle_reply` directly.

A **temporary** pytest harness (`tests/blitzy_adhoc_test_investigation.py`, deleted afterward — see "Clean up") built an `aiosmtpd` `Envelope` + `Message` and called `handle()`. It ran under the `flask_client` fixture (`tests/conftest.py:L60-L77`) inside a Flask app context, mirroring the existing suite's pattern (`tests/test_email_handler.py:L79-L108`). Fixtures were seeded through the app's own APIs — `create_new_user()` (`tests/utils.py:L17`) and `Alias.create_new_random(user)` — and the outbound message was captured with `mail_sender.store_emails_instead_of_sending()` (`app/mail_sender.py:L102`) + `get_stored_emails()` (`app/mail_sender.py:L108`), serialized byte-exact via `message_to_bytes()` (`app/message_utils.py:L12`).

DEBUG logging is always on (`app/log.py:L51` sets the `SL` logger to `DEBUG`). To capture each `LOG.d(...)` line **with its exact `SL` prefix** and without interleaving, the harness attached a `StreamHandler` using the application's own log format (`app/log.py:L12-L15`) with `converter = time.gmtime`, and read the captured buffer for each `handle()` call:

```
"%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s"
```

The seed for `run1`/`run2` (fresh alias + verified mailbox per run):


**`run1`:**

```
user='user_2epkb43th4@mailbox.test'  default_mailbox='user_2epkb43th4@mailbox.test' (verified=True)  alias='labile_censor693@sl.local'
Q3 include_sender_in_reverse_alias = True
```

**`run2`:**

```
user='user_98swtciuky@mailbox.test'  default_mailbox='user_98swtciuky@mailbox.test' (verified=True)  alias='batter_lumped442@sl.local'
Q3 include_sender_in_reverse_alias = True
```

The producing command for every code block in Q1–Q4 (run twice, as `run1`/`run2`):

```bash
# inside container sl-setup
source /tmp/sl_env.sh                                   # venv + PYTHONPATH + CONFIG=tests/test.env + MEM_STORE_URI=redis://localhost
export DB_URI=postgresql://test:test@localhost:15432/test   # canonical PostgreSQL 13 (tests/test.env:L17)
cd /app
CONFIG=/app/tests/test.env alembic upgrade head         # provision the PG13 schema (run once, before run1)
unset PYTEST_ADDOPTS
CONFIG=/app/tests/test.env python -m pytest -o addopts="" -p no:rerunfailures \
  --timeout=180 -q -s tests/blitzy_adhoc_test_investigation.py
```

### DB state, before/during/after, and clean-up

The `flask_client` fixture opens `connection.begin()` and calls `transaction.rollback()` at teardown (`tests/conftest.py:L61,L74-L77`). **Observed reality on this canonical PG13 run:** the rows created by a run are fully visible **during** the run (which is what the Q4 answer relies on — the harness reads the freshly-committed IDs/timestamps live within the same Session), and the teardown `rollback()` **leaves the database rows unchanged** — a probe from a fresh connection immediately after `run1` (from a pristine, freshly-migrated DB) found `0` rows in `contact`, `email_log`, and `message_id_matching`. The autoincrement sequences are **not** transactional, so they advance even though the rows are rolled back; that is precisely why `run2` observes higher IDs than `run1` (e.g. `Contact.id` `1 → 2`, forward `EmailLog.id` `1 → 3`) despite each run starting from `max id = 0`. To fully guarantee the environment is left clean regardless of rollback semantics, the **throwaway PostgreSQL 13 instance created for this investigation was dropped entirely afterward** and the temporary harness was deleted (see "Read-only confirmation").

### Read-only confirmation

No source file was modified. The source baseline commit **`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`** is unchanged except for this one added documentation file: `git diff 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c..HEAD --name-status` shows only `A  blitzy/documentation/app_2cd6ee777f8c.md`, and the working tree is clean (`git status --porcelain` is empty). The temporary observation harness was created only inside the throwaway container's checkout (never in the committed repository) and was deleted after the run; the throwaway PostgreSQL 13 instance was dropped. No other containers were created — the warmed `sl-setup` container was reused.

---

## The exact log message text when an email is successfully forwarded versus when it fails due to a non-existent alias.

Both branches were exercised **separately** through `email_handler.handle(...)` with DEBUG on. They are distinct code paths: the success case runs `forward_email_to_mailbox()` (`email_handler.py:L679`) and returns `status.E200` (`email_handler.py:L928`); the non-existent-alias case returns early inside `handle_forward()` (`email_handler.py:L536`) with `status.E515` (`email_handler.py:L555`).

The `SL` logger and `LOG.d` shortcut are defined in `app/log.py`: `LOG = _get_logger("SL")` (`app/log.py:L79`), `logging.Logger.d = logging.Logger.debug` (`app/log.py:L74`), and the format string is at `app/log.py:L12-L15`. Every line below is the **complete, unedited** capture for that single `handle()` call (all `SL` records, including the full `headers:` list emitted at `email_handler.py:L1980`), pasted with the full prefix.

### (a) Successful forward — returns `E200`

Complete, unedited `SL` log output captured for one successful forward — **`run1`**:


```
2026-07-07 19:26:20,605 - SL - INFO - 27339 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-07 19:26:20,605 - SL - DEBUG - 27339 - "/app/app/log.py:24" - set_message_id() -  - set message_id 6D8C13F069
2026-07-07 19:26:20,606 - SL - DEBUG - 27339 - "/app/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:spoofedemailsource@gmail.com, rcpt_tos:['labile_censor693@sl.local'], header_from:spoofedemailsource@gmail.com, header_to:labile_censor693@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'labile_censor693@sl.local'), ('From', 'spoofedemailsource@gmail.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('X-Spamd-Result', 'default: False [0.50 / 13.00];\n        MID_RHS_NOT_FQDN(0.50)[];\n        DMARC_POLICY_SOFTFAIL(0.10)[gmail.com : No valid SPF, No valid DKIM,none];\n        MIME_GOOD(-0.10)[text/plain];\n        MIME_TRACE(0.00)[0:+];\n        FROM_EQ_ENVFROM(0.00)[];\n        ASN(0.00)[asn:16276, ipnet:34.59.0.0/16, country:FR];\n        R_DKIM_NA(0.00)[];\n        RCVD_COUNT_ZERO(0.00)[0];\n        FREEMAIL_ENVFROM(0.00)[gmail.com];\n        FROM_NO_DN(0.00)[];\n        R_SPF_SOFTFAIL(0.00)[~all];\n        FORCE_ACTION_SL_SPF_FAIL_ADD_HEADER(0.00)[add header];\n        RCPT_COUNT_ONE(0.00)[1];\n        FREEMAIL_FROM(0.00)[gmail.com];\n        TO_DN_NONE(0.00)[];\n        TO_MATCH_ENVRCPT_ALL(0.00)[];\n        ARC_NA(0.00)[]'), ('X-Rspamd-Pre-Result', 'action=add header;\n        module=force_actions;\n        unknown reason'), ('X-Spam', 'Yes'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-07 19:26:20,610 - SL - DEBUG - 27339 - "/app/email_handler.py:2202" - handle() - 6D8C13F069 - Forward phase spoofedemailsource@gmail.com(spoofedemailsource@gmail.com) -> labile_censor693@sl.local
2026-07-07 19:26:20,616 - SL - DEBUG - 27339 - "/app/email_handler.py:580" - handle_forward() - 6D8C13F069 - Create or get contact for from_header:spoofedemailsource@gmail.com
2026-07-07 19:26:20,633 - SL - DEBUG - 27339 - "/app/app/contact_utils.py:110" - create_contact() - 6D8C13F069 - Created contact <Contact 1 spoofedemailsource@gmail.com 2> for alias <Alias 2 labile_censor693@sl.local> with email spoofedemailsource@gmail.com invalid_email=False
2026-07-07 19:26:20,634 - SL - INFO - 27339 - "/app/app/handler/dmarc.py:35" - apply_dmarc_policy_for_forward_phase() - 6D8C13F069 - Spam check result in <app.handler.spamd_result.SpamdResult object at 0x79f7f289a6e0>
2026-07-07 19:26:20,634 - SL - WARNING - 27339 - "/app/app/handler/dmarc.py:72" - apply_dmarc_policy_for_forward_phase() - 6D8C13F069 - dmarc forward: soft_fail from contact spoofedemailsource@gmail.com to alias labile_censor693@sl.local.mail_from:spoofedemailsource@gmail.com, from_header: spoofedemailsource@gmail.com
2026-07-07 19:26:20,641 - SL - DEBUG - 27339 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 6D8C13F069 - Forward <Contact 1 spoofedemailsource@gmail.com 2> -> <Alias 2 labile_censor693@sl.local> -> <Mailbox 1 user_2epkb43th4@mailbox.test>
2026-07-07 19:26:20,644 - SL - DEBUG - 27339 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 6D8C13F069 - Create <EmailLog 1> for <Contact 1 spoofedemailsource@gmail.com 2>, <User 1 Test User user_2epkb43th4@mailbox.test>, <Mailbox 1 user_2epkb43th4@mailbox.test>
2026-07-07 19:26:20,649 - SL - DEBUG - 27339 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 6D8C13F069 - From header, new:"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local>, old:spoofedemailsource@gmail.com
2026-07-07 19:26:20,649 - SL - DEBUG - 27339 - "/app/email_handler.py:316" - replace_header_when_forward() - 6D8C13F069 - Delete Cc header, old value None
2026-07-07 19:26:20,649 - SL - DEBUG - 27339 - "/app/email_handler.py:313" - replace_header_when_forward() - 6D8C13F069 - Replace To header, old: labile_censor693@sl.local, new: labile_censor693@sl.local
2026-07-07 19:26:20,649 - SL - INFO - 27339 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 6D8C13F069 - Email has no unsubscribe header
2026-07-07 19:26:20,652 - SL - DEBUG - 27339 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 6D8C13F069 - Forward mail from spoofedemailsource@gmail.com to user_2epkb43th4@mailbox.test, mail_options:[], rcpt_options:[] 
2026-07-07 19:26:20,652 - SL - DEBUG - 27339 - "/app/app/mail_sender.py:131" - send() - 6D8C13F069 - send email with subject '[Possible phishing attempt] test Thu, 17 Mar 2022 16:50:18 +0000', from '"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local>' to 'labile_censor693@sl.local'
```

Returned SMTP status (byte-exact), and its definition — **`run1`**:

```
result = '250 Message accepted for delivery'
status.E200 = '250 Message accepted for delivery'  -> match=True
```

The same successful forward, **`run2`** (complete, unedited):

```
2026-07-07 19:26:23,327 - SL - INFO - 27356 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-07 19:26:23,327 - SL - DEBUG - 27356 - "/app/app/log.py:24" - set_message_id() -  - set message_id 6D8C13F069
2026-07-07 19:26:23,328 - SL - DEBUG - 27356 - "/app/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:spoofedemailsource@gmail.com, rcpt_tos:['batter_lumped442@sl.local'], header_from:spoofedemailsource@gmail.com, header_to:batter_lumped442@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[('X-SimpleLogin-Client-IP', '54.39.200.130'), ('Received-SPF', 'Softfail (mailfrom) identity=mailfrom; client-ip=34.59.200.130;\n helo=relay.somewhere.net; envelope-from=everwaste@gmail.com;\n receiver=<UNKNOWN>'), ('Received', 'from relay.somewhere.net (relay.somewhere.net [34.59.200.130])\n        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))\n        (No client certificate requested)\n        by mx1.sldev.ovh (Postfix) with ESMTPS id 6D8C13F069\n        for <wehrman_mannequin@sldev.ovh>; Thu, 17 Mar 2022 16:50:20 +0000 (UTC)'), ('Date', 'Thu, 17 Mar 2022 16:50:18 +0000'), ('To', 'batter_lumped442@sl.local'), ('From', 'spoofedemailsource@gmail.com'), ('Subject', 'test Thu, 17 Mar 2022 16:50:18 +0000'), ('Message-Id', '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('X-Rspamd-Queue-Id', '6D8C13F069'), ('X-Rspamd-Server', 'staging1'), ('X-Spamd-Result', 'default: False [0.50 / 13.00];\n        MID_RHS_NOT_FQDN(0.50)[];\n        DMARC_POLICY_SOFTFAIL(0.10)[gmail.com : No valid SPF, No valid DKIM,none];\n        MIME_GOOD(-0.10)[text/plain];\n        MIME_TRACE(0.00)[0:+];\n        FROM_EQ_ENVFROM(0.00)[];\n        ASN(0.00)[asn:16276, ipnet:34.59.0.0/16, country:FR];\n        R_DKIM_NA(0.00)[];\n        RCVD_COUNT_ZERO(0.00)[0];\n        FREEMAIL_ENVFROM(0.00)[gmail.com];\n        FROM_NO_DN(0.00)[];\n        R_SPF_SOFTFAIL(0.00)[~all];\n        FORCE_ACTION_SL_SPF_FAIL_ADD_HEADER(0.00)[add header];\n        RCPT_COUNT_ONE(0.00)[1];\n        FREEMAIL_FROM(0.00)[gmail.com];\n        TO_DN_NONE(0.00)[];\n        TO_MATCH_ENVRCPT_ALL(0.00)[];\n        ARC_NA(0.00)[]'), ('X-Rspamd-Pre-Result', 'action=add header;\n        module=force_actions;\n        unknown reason'), ('X-Spam', 'Yes'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-07 19:26:23,331 - SL - DEBUG - 27356 - "/app/email_handler.py:2202" - handle() - 6D8C13F069 - Forward phase spoofedemailsource@gmail.com(spoofedemailsource@gmail.com) -> batter_lumped442@sl.local
2026-07-07 19:26:23,338 - SL - DEBUG - 27356 - "/app/email_handler.py:580" - handle_forward() - 6D8C13F069 - Create or get contact for from_header:spoofedemailsource@gmail.com
2026-07-07 19:26:23,356 - SL - DEBUG - 27356 - "/app/app/contact_utils.py:110" - create_contact() - 6D8C13F069 - Created contact <Contact 2 spoofedemailsource@gmail.com 4> for alias <Alias 4 batter_lumped442@sl.local> with email spoofedemailsource@gmail.com invalid_email=False
2026-07-07 19:26:23,356 - SL - INFO - 27356 - "/app/app/handler/dmarc.py:35" - apply_dmarc_policy_for_forward_phase() - 6D8C13F069 - Spam check result in <app.handler.spamd_result.SpamdResult object at 0x7a9efce1ccd0>
2026-07-07 19:26:23,356 - SL - WARNING - 27356 - "/app/app/handler/dmarc.py:72" - apply_dmarc_policy_for_forward_phase() - 6D8C13F069 - dmarc forward: soft_fail from contact spoofedemailsource@gmail.com to alias batter_lumped442@sl.local.mail_from:spoofedemailsource@gmail.com, from_header: spoofedemailsource@gmail.com
2026-07-07 19:26:23,364 - SL - DEBUG - 27356 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 6D8C13F069 - Forward <Contact 2 spoofedemailsource@gmail.com 4> -> <Alias 4 batter_lumped442@sl.local> -> <Mailbox 2 user_98swtciuky@mailbox.test>
2026-07-07 19:26:23,366 - SL - DEBUG - 27356 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 6D8C13F069 - Create <EmailLog 3> for <Contact 2 spoofedemailsource@gmail.com 4>, <User 2 Test User user_98swtciuky@mailbox.test>, <Mailbox 2 user_98swtciuky@mailbox.test>
2026-07-07 19:26:23,372 - SL - DEBUG - 27356 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 6D8C13F069 - From header, new:"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_tqrdtt@sl.local>, old:spoofedemailsource@gmail.com
2026-07-07 19:26:23,372 - SL - DEBUG - 27356 - "/app/email_handler.py:316" - replace_header_when_forward() - 6D8C13F069 - Delete Cc header, old value None
2026-07-07 19:26:23,372 - SL - DEBUG - 27356 - "/app/email_handler.py:313" - replace_header_when_forward() - 6D8C13F069 - Replace To header, old: batter_lumped442@sl.local, new: batter_lumped442@sl.local
2026-07-07 19:26:23,372 - SL - INFO - 27356 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 6D8C13F069 - Email has no unsubscribe header
2026-07-07 19:26:23,374 - SL - DEBUG - 27356 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 6D8C13F069 - Forward mail from spoofedemailsource@gmail.com to user_98swtciuky@mailbox.test, mail_options:[], rcpt_options:[] 
2026-07-07 19:26:23,375 - SL - DEBUG - 27356 - "/app/app/mail_sender.py:131" - send() - 6D8C13F069 - send email with subject '[Possible phishing attempt] test Thu, 17 Mar 2022 16:50:18 +0000', from '"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_tqrdtt@sl.local>' to 'batter_lumped442@sl.local'
```

Returned SMTP status — **`run2`**:

```
result = '250 Message accepted for delivery'
status.E200 = '250 Message accepted for delivery'  -> match=True
```

`E200 = "250 Message accepted for delivery"` is defined at `app/email/status.py:L2` and returned at `email_handler.py:L928` (`return True, status.E200`).

The three signature success lines, tied to their source, are:

| Log message | Emitting call | Location |
|---|---|---|
| `Forward <Contact ...> -> <Alias ...> -> <Mailbox ...>` | `LOG.d("Forward %s -> %s -> %s", contact, alias, mailbox)` | `email_handler.py:L688` |
| `Create <EmailLog ...> for <Contact ...>, <User ...>, <Mailbox ...>` | `LOG.d("Create %s for %s, %s, %s", email_log, contact, user, mailbox)` | `email_handler.py:L740` |
| `From header, new:..., old:...` | `LOG.d("From header, new:%s, old:%s", new_from_header, old_from_header)` | `email_handler.py:L867` |

### (b) Forward to a non-existent alias — returns `E515`

Complete, unedited output captured for a forward to `ycqazrdhwsugymxcfbpo@sl.local` (an alias that does not exist and cannot be auto-created) — **`run1`**:


```
2026-07-07 19:26:20,664 - SL - DEBUG - 27339 - "/app/email_handler.py:1963" - handle() - 6D8C13F069 - Cannot parse Postfix queue ID from None None
2026-07-07 19:26:20,665 - SL - DEBUG - 27339 - "/app/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com, rcpt_tos:['ycqazrdhwsugymxcfbpo@sl.local'], header_from:crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com, header_to:ycqazrdhwsugymxcfbpo@sl.local, cc:None, reply-to:None, message_id:<nonexist-1@sender.example.com>, client_ip:None, headers:[('From', 'crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com'), ('To', 'ycqazrdhwsugymxcfbpo@sl.local'), ('Subject', 'to a non-existent alias'), ('Message-ID', '<nonexist-1@sender.example.com>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-07 19:26:20,668 - SL - DEBUG - 27339 - "/app/email_handler.py:2202" - handle() - 6D8C13F069 - Forward phase crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com(crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com) -> ycqazrdhwsugymxcfbpo@sl.local
2026-07-07 19:26:20,674 - SL - DEBUG - 27339 - "/app/email_handler.py:545" - handle_forward() - 6D8C13F069 - alias ycqazrdhwsugymxcfbpo@sl.local not exist. Try to see if it can be created on the fly
2026-07-07 19:26:20,679 - SL - INFO - 27339 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 6D8C13F069 - Cannot auto-create custom domain alias for ycqazrdhwsugymxcfbpo@sl.local because there's no custom domain for sl.local
2026-07-07 19:26:20,679 - SL - INFO - 27339 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 6D8C13F069 - Cannot auto-create ycqazrdhwsugymxcfbpo@sl.local since it has no directory separator
2026-07-07 19:26:20,680 - SL - DEBUG - 27339 - "/app/email_handler.py:551" - handle_forward() - 6D8C13F069 - alias ycqazrdhwsugymxcfbpo@sl.local cannot be created on-the-fly, return 550
```

Returned SMTP status (byte-exact), and its definition — **`run1`**:

```
result = '550 SL E515 Email not exist'
status.E515 = '550 SL E515 Email not exist'  -> match=True
```

The same non-existent-alias forward, **`run2`** (complete, unedited):

```
2026-07-07 19:26:23,386 - SL - DEBUG - 27356 - "/app/email_handler.py:1963" - handle() - 6D8C13F069 - Cannot parse Postfix queue ID from None None
2026-07-07 19:26:23,387 - SL - DEBUG - 27356 - "/app/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com, rcpt_tos:['ycqazrdhwsugymxcfbpo@sl.local'], header_from:crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com, header_to:ycqazrdhwsugymxcfbpo@sl.local, cc:None, reply-to:None, message_id:<nonexist-1@sender.example.com>, client_ip:None, headers:[('From', 'crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com'), ('To', 'ycqazrdhwsugymxcfbpo@sl.local'), ('Subject', 'to a non-existent alias'), ('Message-ID', '<nonexist-1@sender.example.com>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-07 19:26:23,390 - SL - DEBUG - 27356 - "/app/email_handler.py:2202" - handle() - 6D8C13F069 - Forward phase crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com(crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com) -> ycqazrdhwsugymxcfbpo@sl.local
2026-07-07 19:26:23,397 - SL - DEBUG - 27356 - "/app/email_handler.py:545" - handle_forward() - 6D8C13F069 - alias ycqazrdhwsugymxcfbpo@sl.local not exist. Try to see if it can be created on the fly
2026-07-07 19:26:23,402 - SL - INFO - 27356 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 6D8C13F069 - Cannot auto-create custom domain alias for ycqazrdhwsugymxcfbpo@sl.local because there's no custom domain for sl.local
2026-07-07 19:26:23,402 - SL - INFO - 27356 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 6D8C13F069 - Cannot auto-create ycqazrdhwsugymxcfbpo@sl.local since it has no directory separator
2026-07-07 19:26:23,402 - SL - DEBUG - 27356 - "/app/email_handler.py:551" - handle_forward() - 6D8C13F069 - alias ycqazrdhwsugymxcfbpo@sl.local cannot be created on-the-fly, return 550
```

Returned SMTP status — **`run2`**:

```
result = '550 SL E515 Email not exist'
status.E515 = '550 SL E515 Email not exist'  -> match=True
```

`E515 = "550 SL E515 Email not exist"` is defined at `app/email/status.py:L51` and returned at `email_handler.py:L555` (`return [(False, status.E515)]`).

The two signature failure lines, tied to their source:

| Log message | Emitting call | Location |
|---|---|---|
| `alias <addr> not exist. Try to see if it can be created on the fly` | `LOG.d("alias %s not exist. Try to see if it can be created on the fly", alias_address)` | `email_handler.py:L545-L548` |
| `alias <addr> cannot be created on-the-fly, return 550` | `LOG.d("alias %s cannot be created on-the-fly, return 550", alias_address)` | `email_handler.py:L551` |

**Run-to-run note (Q1).** The message *text* is stable across both runs; the volatile pieces are the identifiers interpolated into it — the random alias local-part (`labile_censor693` → `batter_lumped442`), the object ids (`<Contact 1 ...>`/`<EmailLog 1>` → `<Contact 2 ...>`/`<EmailLog 3>`), the mailbox address, the reverse-alias suffix (`_vfrjnqnmr` → `_tqrdtt`), and the timestamp/PID (`27339` → `27356`) in the prefix. Both success runs returned `E200`; both failure runs returned `E515` (see the `match=True` lines above).

---

## What specific SL Message-ID gets generated during forwarding, and how it differs from the original Message-ID.

### Direct answer (the observed truth): a forward generates **no** SL Message-ID

On a **forward**, SimpleLogin does **not** mint an "SL Message-ID." The original inbound `Message-ID` is **preserved byte-for-byte** on the outgoing forwarded message, and the `EmailLog.sl_message_id` column stays **`NULL`**. In the forward path, `replace_sl_message_id_by_original_message_id(msg)` (`email_handler.py:L860`, defined at `email_handler.py:L931`) only remaps `In-Reply-To`/`References` — it does **not** touch `Message-ID`, and the original `Message-ID` is retained in `headers_to_keep`.

Captured before/after for the same forward as Q1(a) — **`run1`**:


```
[Q2 BEFORE] inbound  Message-ID = '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'
[Q2 AFTER]  outbound Message-ID = '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'
[Q2] inbound==outbound Message-ID ? True
[Q2] all header names on outbound forward = ['Date', 'Subject', 'Message-Id', 'Content-Transfer-Encoding', 'X-SimpleLogin-Type', 'X-SimpleLogin-EmailLog-ID', 'X-SimpleLogin-Envelope-From', 'X-SimpleLogin-Original-From', 'X-SimpleLogin-Envelope-To', 'From', 'To', 'DKIM-Signature']
```

**`run2`** (complete, unedited):

```
[Q2 BEFORE] inbound  Message-ID = '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'
[Q2 AFTER]  outbound Message-ID = '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'
[Q2] inbound==outbound Message-ID ? True
[Q2] all header names on outbound forward = ['Date', 'Subject', 'Message-Id', 'Content-Transfer-Encoding', 'X-SimpleLogin-Type', 'X-SimpleLogin-EmailLog-ID', 'X-SimpleLogin-Envelope-From', 'X-SimpleLogin-Original-From', 'X-SimpleLogin-Envelope-To', 'From', 'To', 'DKIM-Signature']
```

The `all header names on outbound forward` line above enumerates every header on the outgoing message: it contains the original `Message-Id` and SimpleLogin tracking headers (`X-SimpleLogin-Type: Forward`, `X-SimpleLogin-EmailLog-ID`, `X-SimpleLogin-Envelope-From/To`, `X-SimpleLogin-Original-From`) plus a `DKIM-Signature`, but **no** SL `Message-ID` header of any kind.

The **complete, byte-exact** outbound header section (from `message_to_bytes()` of the stored `SendRequest`, including the full `DKIM-Signature`) confirms the `Message-Id` is unchanged — **`run1`**:


```
Date: Thu, 17 Mar 2022 16:50:18 +0000
Subject: [Possible phishing attempt] test Thu, 17 Mar 2022 16:50:18 +0000
Message-Id: <20220317165018.000191@somewhere-5488dd4b6b-7crp6>
Content-Transfer-Encoding: 7bit
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 1
X-SimpleLogin-Envelope-From: spoofedemailsource@gmail.com
X-SimpleLogin-Original-From: spoofedemailsource@gmail.com
X-SimpleLogin-Envelope-To: labile_censor693@sl.local
From: "spoofedemailsource at gmail.com"
 <spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local>
To: labile_censor693@sl.local
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local;  i=@sl.local;
 q=dns/txt; s=dkim; t=1783452380; h=message-id : date :  subject : from : to;
 bh=DmVjG4ZfnJiR8NgyQhqy4iUCztIP7mSwLMQ4jPy1gug=;
  b=kJfYTyaOIZHhs5G6Ff++gLRWUD5Wv/OnVnsXiReni3lzJguIe98X03vQNkRPcc/HdLNJ/
  ho9/7Zff4S1CdT0MMCMl0nA7enhdeFrAQDQuE08sV39eaHfuxzPSj2BG7wc52zBw/MpGosk
  8gv/NxYQiMahwuBafQBvs2V0ufta7zw= 
```

**`run2`** (complete, byte-exact):

```
Date: Thu, 17 Mar 2022 16:50:18 +0000
Subject: [Possible phishing attempt] test Thu, 17 Mar 2022 16:50:18 +0000
Message-Id: <20220317165018.000191@somewhere-5488dd4b6b-7crp6>
Content-Transfer-Encoding: 7bit
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 3
X-SimpleLogin-Envelope-From: spoofedemailsource@gmail.com
X-SimpleLogin-Original-From: spoofedemailsource@gmail.com
X-SimpleLogin-Envelope-To: batter_lumped442@sl.local
From: "spoofedemailsource at gmail.com"
 <spoofedemailsource_at_gmail_com_tqrdtt@sl.local>
To: batter_lumped442@sl.local
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local;  i=@sl.local;
 q=dns/txt; s=dkim; t=1783452383; h=message-id : date :  subject : from : to;
 bh=DmVjG4ZfnJiR8NgyQhqy4iUCztIP7mSwLMQ4jPy1gug=;
  b=APCZ+CuUcPAYfSINhBPdG4fcIH6JY8At1/yHEgUgPlMtW6DscCQSX+7M2gQq0TUxQaUrx
  Hx+9d8eQlZCVwTL2G2c2fz8YjC+aSerC6RDeUpKPErSxICb6fqnTwzdLX53hS3tHAyVZTIg
  1PVFXvryHDAmF9aa1dEJf5ygdZ/Lj+w= 
```

And the created `EmailLog` row confirms `sl_message_id` is `NULL` on a forward (see the `EMAILLOG ... sl_message_id=None` line in Q4 below, and the `new MessageIDMatching rows = 0` count). `EmailLog.sl_message_id` is a nullable column (`app/models.py:L2116`, deferred `String(512)`) documented as "in the reply phase, the original message_id is replaced by the SL message_id." On a forward it is never written. **No `MessageIDMatching` row is created on a forward** either (`MessageIDMatching` defined at `app/models.py:L3365`).

### For contrast: a REPLY does mint an SL Message-ID

The "SL Message-ID" the question references is a **reply-phase** construct. Exercising a reply (rcpt = the reverse-alias, `mail_from` = the alias's own mailbox) routes to `handle_reply()` (`email_handler.py:L966`), which calls `replace_original_message_id(alias, email_log, msg)` (`email_handler.py:L1202`; defined at `email_handler.py:L1296`). That function mints the SL Message-ID at `email_handler.py:L1311-L1313` and logs it at `email_handler.py:L1314`:

```python
sl_message_id = make_msgid(
    str(email_log.id), get_email_domain_part(alias.email)
)
LOG.d("create a new sl_message_id %s", sl_message_id)
```

Complete, unedited log captured from the reply — **`run1`**:


```
2026-07-07 19:26:20,682 - SL - DEBUG - 27339 - "/app/email_handler.py:1963" - handle() - 6D8C13F069 - Cannot parse Postfix queue ID from None None
2026-07-07 19:26:20,683 - SL - DEBUG - 27339 - "/app/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:user_2epkb43th4@mailbox.test, rcpt_tos:['spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local'], header_from:user_2epkb43th4@mailbox.test, header_to:spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local, cc:None, reply-to:None, message_id:<orig-reply-1@mailbox.test>, client_ip:None, headers:[('From', 'user_2epkb43th4@mailbox.test'), ('To', 'spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local'), ('Subject', 'Re: reply through reverse-alias'), ('Message-ID', '<orig-reply-1@mailbox.test>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-07 19:26:20,686 - SL - DEBUG - 27339 - "/app/email_handler.py:2196" - handle() - 6D8C13F069 - Reply phase user_2epkb43th4@mailbox.test(user_2epkb43th4@mailbox.test) -> spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local
2026-07-07 19:26:20,688 - SL - INFO - 27339 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 6D8C13F069 - DMARC check disabled
2026-07-07 19:26:20,690 - SL - DEBUG - 27339 - "/app/email_handler.py:1051" - handle_reply() - 6D8C13F069 - Create <EmailLog 2> for <Contact 1 spoofedemailsource@gmail.com 2>, <User 1 Test User user_2epkb43th4@mailbox.test>, <Mailbox 1 user_2epkb43th4@mailbox.test>
2026-07-07 19:26:20,695 - SL - DEBUG - 27339 - "/app/email_handler.py:1171" - handle_reply() - 6D8C13F069 - From header is labile_censor693@sl.local
2026-07-07 19:26:20,696 - SL - DEBUG - 27339 - "/app/email_handler.py:380" - replace_header_when_reply() - 6D8C13F069 - Replace To header, old: spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local, new: spoofedemailsource@gmail.com
2026-07-07 19:26:20,697 - SL - DEBUG - 27339 - "/app/email_handler.py:383" - replace_header_when_reply() - 6D8C13F069 - delete the Cc header. Old value None
2026-07-07 19:26:20,698 - SL - DEBUG - 27339 - "/app/email_handler.py:1314" - replace_original_message_id() - 6D8C13F069 - create a new sl_message_id <178345238069.27339.1560924962577896419.2@sl.local>
2026-07-07 19:26:20,701 - SL - WARNING - 27339 - "/app/email_handler.py:1206" - handle_reply() - 6D8C13F069 - missing date header, add one
2026-07-07 19:26:20,704 - SL - DEBUG - 27339 - "/app/email_handler.py:1212" - handle_reply() - 6D8C13F069 - send email from labile_censor693@sl.local to spoofedemailsource@gmail.com, mail_options:[],rcpt_options:[]
2026-07-07 19:26:20,707 - SL - DEBUG - 27339 - "/app/app/mail_sender.py:131" - send() - 6D8C13F069 - send email with subject 'Re: reply through reverse-alias', from 'labile_censor693@sl.local' to 'spoofedemailsource@gmail.com'
```

The minted value and the DB rows it produces — **`run1`**:

```
result = '250 Message accepted for delivery' (E200 match=True)
[Q2-reply BEFORE] inbound reply Message-ID = '<orig-reply-1@mailbox.test>'
[Q2-reply] reply EmailLog.id=2  EmailLog.sl_message_id='<178345238069.27339.1560924962577896419.2@sl.local>'
[Q2-reply] MessageIDMatching id=1 created_at='2026-07-07T19:26:20.699071+00:00'
           sl_message_id       = '<178345238069.27339.1560924962577896419.2@sl.local>'
           original_message_id = '<orig-reply-1@mailbox.test>'
           email_log_id        = 2
[Q2-reply AFTER] outbound reply Message-ID = '<178345238069.27339.1560924962577896419.2@sl.local>'
```

The same reply, **`run2`** (complete, unedited log):

```
2026-07-07 19:26:23,404 - SL - DEBUG - 27356 - "/app/email_handler.py:1963" - handle() - 6D8C13F069 - Cannot parse Postfix queue ID from None None
2026-07-07 19:26:23,405 - SL - DEBUG - 27356 - "/app/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:user_98swtciuky@mailbox.test, rcpt_tos:['spoofedemailsource_at_gmail_com_tqrdtt@sl.local'], header_from:user_98swtciuky@mailbox.test, header_to:spoofedemailsource_at_gmail_com_tqrdtt@sl.local, cc:None, reply-to:None, message_id:<orig-reply-1@mailbox.test>, client_ip:None, headers:[('From', 'user_98swtciuky@mailbox.test'), ('To', 'spoofedemailsource_at_gmail_com_tqrdtt@sl.local'), ('Subject', 'Re: reply through reverse-alias'), ('Message-ID', '<orig-reply-1@mailbox.test>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-07 19:26:23,409 - SL - DEBUG - 27356 - "/app/email_handler.py:2196" - handle() - 6D8C13F069 - Reply phase user_98swtciuky@mailbox.test(user_98swtciuky@mailbox.test) -> spoofedemailsource_at_gmail_com_tqrdtt@sl.local
2026-07-07 19:26:23,411 - SL - INFO - 27356 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 6D8C13F069 - DMARC check disabled
2026-07-07 19:26:23,413 - SL - DEBUG - 27356 - "/app/email_handler.py:1051" - handle_reply() - 6D8C13F069 - Create <EmailLog 4> for <Contact 2 spoofedemailsource@gmail.com 4>, <User 2 Test User user_98swtciuky@mailbox.test>, <Mailbox 2 user_98swtciuky@mailbox.test>
2026-07-07 19:26:23,418 - SL - DEBUG - 27356 - "/app/email_handler.py:1171" - handle_reply() - 6D8C13F069 - From header is batter_lumped442@sl.local
2026-07-07 19:26:23,420 - SL - DEBUG - 27356 - "/app/email_handler.py:380" - replace_header_when_reply() - 6D8C13F069 - Replace To header, old: spoofedemailsource_at_gmail_com_tqrdtt@sl.local, new: spoofedemailsource@gmail.com
2026-07-07 19:26:23,420 - SL - DEBUG - 27356 - "/app/email_handler.py:383" - replace_header_when_reply() - 6D8C13F069 - delete the Cc header. Old value None
2026-07-07 19:26:23,421 - SL - DEBUG - 27356 - "/app/email_handler.py:1314" - replace_original_message_id() - 6D8C13F069 - create a new sl_message_id <178345238342.27356.3145874089878559383.4@sl.local>
2026-07-07 19:26:23,425 - SL - WARNING - 27356 - "/app/email_handler.py:1206" - handle_reply() - 6D8C13F069 - missing date header, add one
2026-07-07 19:26:23,428 - SL - DEBUG - 27356 - "/app/email_handler.py:1212" - handle_reply() - 6D8C13F069 - send email from batter_lumped442@sl.local to spoofedemailsource@gmail.com, mail_options:[],rcpt_options:[]
2026-07-07 19:26:23,431 - SL - DEBUG - 27356 - "/app/app/mail_sender.py:131" - send() - 6D8C13F069 - send email with subject 'Re: reply through reverse-alias', from 'batter_lumped442@sl.local' to 'spoofedemailsource@gmail.com'
```

Minted value and DB rows — **`run2`**:

```
result = '250 Message accepted for delivery' (E200 match=True)
[Q2-reply BEFORE] inbound reply Message-ID = '<orig-reply-1@mailbox.test>'
[Q2-reply] reply EmailLog.id=4  EmailLog.sl_message_id='<178345238342.27356.3145874089878559383.4@sl.local>'
[Q2-reply] MessageIDMatching id=2 created_at='2026-07-07T19:26:23.422473+00:00'
           sl_message_id       = '<178345238342.27356.3145874089878559383.4@sl.local>'
           original_message_id = '<orig-reply-1@mailbox.test>'
           email_log_id        = 4
[Q2-reply AFTER] outbound reply Message-ID = '<178345238342.27356.3145874089878559383.4@sl.local>'
```

So on a reply, unlike a forward: the original `Message-ID` (`<orig-reply-1@mailbox.test>`) is **replaced** on the outgoing message by the minted SL Message-ID (`email_handler.py:L1338-L1339`), `EmailLog.sl_message_id` is **set** (`email_handler.py:L1341`), and a `MessageIDMatching` row is created (`email_handler.py:L1316-L1321`) recording the `sl_message_id`↔`original_message_id`↔`email_log_id` mapping.

**Structure of the minted SL Message-ID** — it is exactly the Python stdlib `email.utils.make_msgid(idstring, domain)` shape `<{timeval}.{pid}.{randint}.{idstring}@{domain}>`. Decoding the observed `run1` value `<178345238069.27339.1560924962577896419.2@sl.local>`:

| Component | Observed (`run1`) | Observed (`run2`) | Meaning |
|---|---|---|---|
| `timeval` | `178345238069` | `178345238342` | stdlib time component |
| `pid` | `27339` | `27356` | process id (matches the `%(process)d` field in every log prefix above) |
| `randint` | `1560924962577896419` | `3145874089878559383` | stdlib random component |
| `idstring` | `2` | `4` | `str(email_log.id)` — the reply `EmailLog.id` |
| `domain` | `sl.local` | `sl.local` | `get_email_domain_part(alias.email)` — the alias domain |

**Run-to-run note (Q2).** The forward preserved the same original `Message-ID` on both runs (nothing minted; `sl_message_id=None`). The reply minted a different value each run (see the two complete reply blocks above). Thus `timeval`, `randint`, and the `email_log.id` component vary by nature; `pid` is stable within a process; the `@sl.local` domain is stable. There is also a reuse branch (`LOG.d("reuse the sl_message_id %s", ...)`, `email_handler.py:L1308-L1309`) for a reply fanned out to multiple recipients — not exercised here.

### Why this is the likely cause of "inconsistent behavior"

A user who inspects a **forwarded** email sees the sender's original `Message-ID` unchanged, while a user who inspects a **reply** they sent through the reverse-alias sees a SimpleLogin-minted `<...@sl.local>` `Message-ID`. Same alias, two phases, two different `Message-ID` outcomes — the forward-vs-reply asymmetry is documented here as the probable explanation. **No change was made** (diagnostic-only).

---

## The exact From header value in the forwarded email after transformation, including the reply-email (reverse-alias) address format.

### The transformed `From` (observed, byte-exact)

In the forward path the `From` header is replaced at `email_handler.py:L862-L867`:

```python
old_from_header = msg[headers.FROM]
new_from_header = contact.new_addr()
add_or_replace_header(msg, "From", new_from_header)
LOG.d("From header, new:%s, old:%s", new_from_header, old_from_header)
```

The captured `From` transformation (before/after, the bare reverse-alias, and `Contact.new_addr()`) — **`run1`**:


```
[Q3 BEFORE] inbound  From = 'spoofedemailsource@gmail.com'
[Q3 AFTER]  outbound From = '"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local>'
[Q3] bare reverse-alias reply_email = 'spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local'
[Q3] Contact.new_addr() = '"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local>'
```

**`run2`** (complete, unedited):

```
[Q3 BEFORE] inbound  From = 'spoofedemailsource@gmail.com'
[Q3 AFTER]  outbound From = '"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_tqrdtt@sl.local>'
[Q3] bare reverse-alias reply_email = 'spoofedemailsource_at_gmail_com_tqrdtt@sl.local'
[Q3] Contact.new_addr() = '"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_tqrdtt@sl.local>'
```

The same value appears byte-exact in the outbound header section above (RFC 5322 header folding across two physical lines), e.g. `run1`:

```
From: "spoofedemailsource at gmail.com"
 <spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local>
```

**So the transformed `From` is:** a quoted display name `"spoofedemailsource at gmail.com"` followed by the reverse-alias in angle brackets. This is produced by `Contact.new_addr()` (`app/models.py:L2008`) with the **default** `SenderFormatEnum.AT` (`= 0`, `app/models.py:L204`; the `User.sender_format` default is `0`). For the `AT` format, `new_addr()` computes `formatted_email = self.website_email.replace("@", " at ").strip()` → `spoofedemailsource at gmail.com`, and because the contact has no distinct display name it uses that as the whole `new_name`, then returns `sl_formataddr((new_name, self.reply_email))` (`app/models.py:L2044`). `sl_formataddr()` (`app/email_utils.py:L1501`) wraps `email.utils.formataddr` with `Header(addr, "utf-8")` to yield RFC-2047 form. Here the value is pure ASCII, so **no** `=?utf-8?...?=` encoding appears; a non-ASCII display name would be RFC-2047 encoded. (Had the contact carried a display name distinct from its email, the `AT` branch would render `"<Name> - spoofedemailsource at gmail.com" <reply_email>` — `app/models.py:L2028-L2033`.)

### The reverse-alias (`reply_email`) address format — observed

The bare reverse-alias placed in `From`/used as `reply_email` is generated by `generate_reply_email(contact_email, alias)` (`app/email_utils.py:L1103`). The observed values are the `[Q3] bare reverse-alias reply_email` lines above: `run1` → `spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local`, `run2` → `spoofedemailsource_at_gmail_com_tqrdtt@sl.local`.

**Important observed detail (actual value vs. the pure-random assumption).** The observed reverse-alias uses the **sender-included** format `{sanitized-sender}_{random}@sl.local`, *not* a pure-random `{random}@sl.local`. The cause, verified at runtime, is that `User.include_sender_in_reverse_alias` has a Python-side ORM **`default=True`** (`app/models.py:L455-L457`: `sa.Column(sa.Boolean, default=True, nullable=False, server_default="0")`). A user created through the ORM (which is how SimpleLogin creates every user, including `create_new_user()`) therefore gets `True`; the `server_default="0"` would only apply to rows inserted via raw SQL outside the ORM. This was observed directly in the `SEED` block for each run above: `Q3 include_sender_in_reverse_alias = True`.

With that flag `True`, `generate_reply_email()` takes the sender-included branch (`app/email_utils.py:L1119-L1141`): the contact email is sanitized (`@` → `_at_`, `.` → `_`, truncated to 45 chars) and suffixed with `random_string(random.randint(5, 10))` at `@{reply_domain}` where `reply_domain = config.EMAIL_DOMAIN = sl.local`. Hence `spoofedemailsource@gmail.com` → `spoofedemailsource_at_gmail_com` + `_` + a random suffix (`vfrjnqnmr`, 9 chars, in `run1`; `tqrdtt`, 6 chars, in `run2`) + `@sl.local`.

For completeness: the **other** branch of `generate_reply_email()` — taken only when `include_sender_in_reverse_alias` is `False` — produces the pure-random `f"{random_string(random.randint(20, 50))}@{reply_domain}"` form (`app/email_utils.py:L1143-L1148`). The legacy `ra+`/`reply+` prefixes are commented out in both branches. This alternate form was **not** observed under the canonical default and is noted here as context only.

**Run-to-run note (Q3).** The display-name portion (`"spoofedemailsource at gmail.com"`) and the sanitized-sender prefix + `@sl.local` domain are stable; the random suffix varies each generation: `run1` `_vfrjnqnmr` (9 chars), `run2` `_tqrdtt` (6 chars) — consistent with `random.randint(5, 10)`.

---

## What database records are created during a single forward operation — show actual record IDs and timestamps.

### Method (before / during / after)

Inside the run, the harness snapshotted `max(id)` of `EmailLog`, `Contact`, `UserAuditLog`, and `MessageIDMatching` **before** the `handle()` call, ran one forward from a **new** sender, then diffed **after**. Every model inherits `ModelMixin` (`app/models.py:L62-L65`), which provides the autoincrement integer `id` and the `created_at` `ArrowType` defaulting to `arrow.utcnow` (`app/models.py:L64`).

### Result — a single new-sender forward creates exactly three rows

Captured before/after — **`run1`** (from a pristine, freshly-migrated DB):


```
[Q4 BEFORE] max Contact.id=0  max EmailLog.id=0  max UserAuditLog.id=0  max MessageIDMatching.id=0

[Q4 AFTER]
new Contact rows = 1, new EmailLog rows = 1, new UserAuditLog rows = 1, new MessageIDMatching rows = 0
  CONTACT      id=1 created_at='2026-07-07T19:26:20.627071+00:00' website_email='spoofedemailsource@gmail.com' reply_email='spoofedemailsource_at_gmail_com_vfrjnqnmr@sl.local'
  EMAILLOG     id=1 created_at='2026-07-07T19:26:20.642466+00:00' is_reply=False blocked=False message_id='<20220317165018.000191@somewhere-5488dd4b6b-7crp6>' sl_message_id=None
  USERAUDITLOG id=1 created_at='2026-07-07T19:26:20.631629+00:00' action='create_contact' message='Created contact 1 (spoofedemailsource@gmail.com)'
  MESSAGEIDMATCHING new rows = 0 (expected 0 on forward)
```

**`run2`** (complete; IDs advanced because the autoincrement sequences are non-transactional — see "DB state" above):

```
[Q4 BEFORE] max Contact.id=0  max EmailLog.id=0  max UserAuditLog.id=0  max MessageIDMatching.id=0

[Q4 AFTER]
new Contact rows = 1, new EmailLog rows = 1, new UserAuditLog rows = 1, new MessageIDMatching rows = 0
  CONTACT      id=2 created_at='2026-07-07T19:26:23.348451+00:00' website_email='spoofedemailsource@gmail.com' reply_email='spoofedemailsource_at_gmail_com_tqrdtt@sl.local'
  EMAILLOG     id=3 created_at='2026-07-07T19:26:23.365080+00:00' is_reply=False blocked=False message_id='<20220317165018.000191@somewhere-5488dd4b6b-7crp6>' sl_message_id=None
  USERAUDITLOG id=2 created_at='2026-07-07T19:26:23.353778+00:00' action='create_contact' message='Created contact 2 (spoofedemailsource@gmail.com)'
  MESSAGEIDMATCHING new rows = 0 (expected 0 on forward)
```

The three created rows (`run1`), each tied to the code that creates it:

| # | Table / model | Actual id (`run1` / `run2`) | Actual `created_at` (`run1`) | Created by |
|---|---|---|---|---|
| 1 | `Contact` (`app/models.py:L1863`) | `1` / `2` | `2026-07-07T19:26:20.627071+00:00` | `create_contact()` (`app/contact_utils.py:L42`); `reply_email` from `generate_reply_email()` (`app/contact_utils.py:L89`) |
| 2 | `UserAuditLog` (`app/models.py:L3829`, table `user_audit_log`) | `1` / `2` | `2026-07-07T19:26:20.631629+00:00` | `emit_user_audit_log(...)` (`app/contact_utils.py:L104-L108`) → `UserAuditLog.create(...)`; `action='create_contact'` |
| 3 | `EmailLog` (`app/models.py:L2060`, table `email_log`) | `1` / `3` | `2026-07-07T19:26:20.642466+00:00` | `EmailLog.create(...)` per verified mailbox (`email_handler.py:L732`), `message_id=str(msg[headers.MESSAGE_ID])` |

Key attributes on the created `EmailLog`: `is_reply=False`, `blocked=False`, `message_id` = the preserved original `Message-ID`, and **`sl_message_id=None`**. As shown, **no `MessageIDMatching`** row is created on a forward (contrast Q2's reply, which created `MessageIDMatching id=1`/`id=2`).

The `created_at` ordering within the operation reflects the code order: the `Contact` (`...627071`) and its `UserAuditLog` (`...631629`) are written first in `handle_forward()`, then the `EmailLog` (`...642466`) is written in `forward_email_to_mailbox()`.

### Run-to-run note (Q4)

Both runs create exactly `1 Contact + 1 UserAuditLog + 1 EmailLog` and `0 MessageIDMatching` on the forward. The autoincrement `id` values and `created_at` timestamps are **volatile by nature** (they were captured, never fabricated); across the two runs `Contact.id` went `1 → 2`, forward `EmailLog.id` `1 → 3` (id 2 was consumed by `run1`'s reply), and `UserAuditLog.id` `1 → 2` — the sequence advance explained in "DB state" above.

### Context: forward from an **already-known** sender

The three-row result is the **new-sender** case. A forward whose sender is already a `Contact` of that alias creates **only** the `EmailLog` (no new `Contact`, no new `UserAuditLog`), because `handle_forward()` uses get-or-create for the contact. The primary answer for "a single forward operation" from a new sender is therefore **1 `Contact` + 1 `UserAuditLog` + 1 `EmailLog`**.

---

## Coverage & caveats

**Both Q1 branches were run and captured separately, for both runs.** Success → `E200` (`"250 Message accepted for delivery"`, `app/email/status.py:L2`); non-existent alias → `E515` (`"550 SL E515 Email not exist"`, `app/email/status.py:L51`). A reply was additionally exercised for the Q2 contrast (minted SL Message-ID + `MessageIDMatching`). Every observation was repeated at least twice and the **complete, unedited output of both runs is pasted above**.

**Volatile-but-captured fields (never fabricated).** These vary by nature run to run and were reproduced, not invented:

- Random alias local-parts (`labile_censor693` in `run1`, `batter_lumped442` in `run2`).
- Reverse-alias random suffix from `random_string(random.randint(5, 10))` (`_vfrjnqnmr`, `_tqrdtt`).
- `make_msgid` components in the reply SL Message-ID: `timeval` (`178345238069` → `178345238342`), `pid` (`27339` → `27356`), `randint` (`1560924962577896419` → `3145874089878559383`), and the `email_log.id` idstring (`2` → `4`).
- Autoincrement `id`s (Contact `1 → 2`; forward EmailLog `1 → 3`; UserAuditLog `1 → 2`; MessageIDMatching `1 → 2`) and `arrow.utcnow` `created_at` timestamps.
- The `DKIM-Signature` `t=` timestamp (`t=1783452380` → `t=1783452383`) and the resulting `b=` signature bytes; the body hash `bh=` is identical across runs because the signed body is unchanged.

Stable structure across runs: the log message text, the `E200`/`E515` status strings, the preserved forward `Message-ID` (`sl_message_id=None`), the `AT`-format display name, the `@sl.local` reverse-alias domain, the outbound header-name set, and the 3-row DB result.

**Canonical runtime.** All values were produced against the canonical stack — Python 3.10.18 and **PostgreSQL 13.23 on port `15432`** (matching `postgres:13` at `.github/workflows/main.yml:L47` and `DB_URI` at `tests/test.env:L17`), with SQLAlchemy 1.3.24, Flask 1.1.2, aiosmtpd 1.4.2, arrow 0.16.0 (see the `RUNTIME` block). No non-canonical substitutions were made.

**DB left unchanged / clean-up.** As detailed in "Environment & method," the `flask_client` fixture's `transaction.rollback()` leaves the DB rows unchanged (verified `0` rows after `run1`), while the non-transactional autoincrement sequences advance. The freshly-committed IDs/timestamps were read live during each run; the throwaway PostgreSQL 13 instance was then dropped entirely, and the temporary harness deleted, so both the repository and the environment are left in their original state.

**Adjacent branches noted as context only (not exercised as answers).** These are alternative outcomes the code can produce but are outside the two Q1 conditions asked about:

- A disabled alias or `contact.block_forward` produces a **blocked** `EmailLog` instead of a delivery (`email_handler.py:L596-L604`).
- An unverified mailbox → `E517` (`app/email/status.py:L53`); a disabled mailbox → `E518` (`app/email/status.py:L54`).
- The observed forward's `Subject` gained a `[Possible phishing attempt]` prefix because the chosen `.eml` fixture (`tests/example_emls/dmarc_gmail_softfail.eml`) carries a DMARC soft-fail rspamd result; this is orthogonal to the four questions (it affects `Subject`, not the `From`/`Message-ID`/records behavior reported here).
- A reply fanned out to multiple recipients reuses an existing `sl_message_id` via the branch at `email_handler.py:L1306-L1309` (`LOG.d("reuse the sl_message_id %s", ...)`).

**Grounding.** Every factual claim above cites a `file:line` locator and names the responsible function/method; every value was produced by running `email_handler.handle(...)` (the real entry point) on the canonical PostgreSQL 13 stack, with the exact producing command shown in "Environment & method." Nothing here was derived from static reading alone.
