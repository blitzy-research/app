# Diagnostic Report — Intermittent "reply routed to the wrong user" in the alias reply pipeline

**Repository:** SimpleLogin (`app`) · **Source-branch commit under investigation:** `app_2cd6ee777f8c` @ `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` · **Working branch:** `blitzy-21ffb38f-7c02-40ed-ae41-d6a514f6966c` (every source/test/config/migration file at the working branch is byte-identical to the source-branch commit; this document is the sole addition)
**Task type:** Read-only, runtime-grounded root-cause investigation (SWE-AtlasQnA-Repo). No source/test/config/migration file was modified; the only durable artifact is this document.

---

## User scenario under investigation

> "When a user replies to an email that was forwarded through an alias, the backend receives the inbound message, identifies which alias it belongs to, and relays it back to the correct recipient. In our local tests, however, we've noticed that some replies appear to be routed to the wrong user, even though the logs correctly show the alias being recognized."

## TL;DR — the finding

The reply pipeline resolves **every** downstream identity (alias, user, authorizing mailbox, the logged `user_id`, and the delivery recipient) from a **single `Contact` row** fetched at:

```python
# email_handler.py:986
    contact = Contact.get_by(reply_email=reply_email)
```

`Contact.get_by` is the generic mixin helper, which is a `filter_by(...).first()` **with no `ORDER BY`**:

```python
# app/models.py:83-84
    def get_by(cls, **kw):
        return Session.query(cls).filter_by(**kw).first()
```

The `reply_email` column carries only a **non-unique** index (no `UniqueConstraint`):

```python
# app/models.py:1899
    reply_email = sa.Column(sa.String(512), nullable=False, index=True)
```

When two `Contact` rows share one `reply_email`, the lookup at `email_handler.py:986` may return **either** row, and SQL does not guarantee which. Because the alias, user, mailbox, logged `user_id`, and recipient are **all derived from that one row**, a wrong-row selection mis-routes the **entire** reply — while the alias is still "recognized" (dispatch's `is_reverse_alias()` succeeds on the *existence* of either duplicate). This is the **most likely origin** of the reported symptom, and it was **reproduced live** below: the identical inbound message routed to **user 829** in one DB/plan state and to **user 830 (the wrong user)** in another, with no change to the input.

> **This is fundamentally a CONFIDENTIALITY defect, not merely a mis-delivery bug.** A reverse-alias is designed to map **1:1** to a single `(alias, contact)` pair precisely so that each user's mailbox and correspondents stay private. When `:986` picks the wrong duplicate, the reply is relayed to a **different** SimpleLogin user's contact (`contact.website_email`, `email_handler.py:1121`) and the `EmailLog` is committed under that **other** user's `user_id` (`email_handler.py:1046`). The private content of one user's reply — and the fact of their correspondence — is thereby disclosed to an unintended recipient belonging to another account, and that other account is credited with sending it. The non-unique `reply_email` column breaks the 1:1 guarantee that keeps users isolated.

> **Note on non-determinism (honest characterization):** the behavior is **order-unspecified**, not application-random. Within a fixed physical (heap) order and query plan the lookup is 100% stable per call; it **flips** across routine writes that change physical row order (MVCC relocation on `UPDATE`, insertion order, `VACUUM`) or across a change of query plan (sequential scan vs. index scan). See section (e).

---

## Methodology (run-first)

Every behavioral claim below is backed by (1) a `file:line` reference, (2) the exact command that produced the evidence, and (3) the complete, unedited captured output. The investigation **ran the code first**, then this document was written from what was observed.

**Canonical runtime** (the agent-host shell's Python 3.12.3 is **non-canonical** and was **not** used for any observed value). All commands ran inside the canonical Docker stack:

- `sl-app` — image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, `--network host`, host repo bind-mounted at `/app`, canonical Python 3.10.18 at `/app/venv/bin/python`.
- `sl-test-db` — `postgres:13` (13.23), host port `15432` → `5432`, db/user/pass = `test`/`test`/`test`.
- `sl-redis` — `redis:6` (6.2.22), port `6379`.

**Canonical invocation pattern** (used throughout):

```bash
docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env /app/venv/bin/<tool> ...'
```

`CONFIG=tests/test.env` is required because `app/config.py` loads that env file at import time; it selects the **default** configuration (`EMAIL_DOMAIN=sl.local`). Temporary observation scripts lived under `/tmp` (outside the repo) and were removed afterward; seeded DB rows were deleted afterward (section (h)).

---

## (a) Exact build & invocation commands used

The stack runs entirely inside the canonical Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container `sl-app`), with PostgreSQL 13 (`sl-test-db`) and Redis 6 (`sl-redis`) alongside. The interpreter and all dependencies live in a virtualenv at `/app/venv`, and the schema is applied with `alembic upgrade head`. Every command below is the exact invocation used, shown with its complete, unedited output.

**Canonical interpreter (Python 3.10, per `pyproject.toml:61` `python = "^3.10"` and `Dockerfile:8` `FROM python:3.10`):**

```bash
docker exec sl-app bash -lc '/app/venv/bin/python --version'
```
```
Python 3.10.18
```

**Dependency provisioning — reconciliation of the canonical container's `/app/venv` against the AAP's `poetry install` (AAP §0.4):** The AAP describes provisioning via `poetry install` against the pinned `poetry.lock`. In the canonical image, `poetry` is **not** on `PATH`; the identical pinned set was instead materialized into a standard pip virtualenv at `/app/venv`. The reply path is therefore exercised through `/app/venv/bin/python`, never the bare system interpreter. Four commands establish this precisely:

*(1) `poetry` is absent from the canonical container:*
```bash
docker exec sl-app bash -lc 'command -v poetry || echo "poetry: not installed in canonical container"'
```
```
poetry: not installed in canonical container
```

*(2) the venv is isolated from system site-packages (its marker file):*
```bash
docker exec sl-app bash -lc 'cat /app/venv/pyvenv.cfg'
```
```
home = /usr/local/bin
include-system-site-packages = false
version = 3.10.18
```

*(3) the bare system interpreter is dependency-less — proving the reply path only runs via `/app/venv` (a bare `import sqlalchemy` fails):*
```bash
docker exec sl-app bash -lc '/usr/local/bin/python -c "import sqlalchemy" 2>&1 | tail -1'
```
```
ModuleNotFoundError: No module named 'sqlalchemy'
```

*(4) the venv imports every reply-path dependency successfully:*
```bash
docker exec sl-app bash -lc '/app/venv/bin/python -c "import flask,sqlalchemy,aiosmtpd,flanker,dkim,dns,psycopg2,arrow,redis,email_validator,alembic; print(\"all reply-path deps importable in /app/venv\")"'
```
```
all reply-path deps importable in /app/venv
```

**Dependency pins (installed in `/app/venv`; each version equals the pin in `poetry.lock` and AAP §0.4):**

```bash
docker exec sl-app bash -lc '/app/venv/bin/pip freeze | grep -iE "^(aiosmtpd|alembic|arrow|dkimpy|dnspython|email-validator|flanker|Flask|psycopg2-binary|redis|SQLAlchemy)=="'
```
```
aiosmtpd==1.4.2
alembic==1.4.3
arrow==0.16.0
dkimpy==1.0.5
dnspython==2.0.0
email-validator==1.1.3
flanker==0.9.11
Flask==1.1.2
psycopg2-binary==2.9.3
redis==4.6.0
SQLAlchemy==1.3.24
```

**PostgreSQL 13 and Redis 6 provisioning verified:**

```bash
docker exec sl-test-db psql -U test -d test -tAc "SHOW server_version;"
```
```
13.23 (Debian 13.23-1.pgdg13+1)
```
```bash
docker exec sl-redis redis-cli INFO server | grep redis_version
```
```
redis_version:6.2.22
```

**Default configuration — `EMAIL_DOMAIN=sl.local` (mechanism: `app/config.py` reads `os.environ["EMAIL_DOMAIN"].lower()`; value supplied by `tests/test.env:8`):**

```bash
docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env /app/venv/bin/python -c "import app.config as c; print(repr(c.EMAIL_DOMAIN))" 2>&1'
```
```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ubgeetujfocbppavlgpl
Upload files to local dir
'sl.local'
```
(The `GNUPGHOME` temp-dir path is init noise that varies per run; the meaningful last line is `'sl.local'`.)

**Schema applied with `alembic upgrade head` (the exact command; complete unedited output).** The DB is already at head, so the run is idempotent and applies no further migrations — its full output is:

```bash
docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env /app/venv/bin/alembic upgrade head 2>&1'
```
```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/fzfumxhxluzyqwhjltyh
Upload files to local dir
>>> init logging <<<
2026-07-08 05:44:52,642 - SL - DEBUG - 14591 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
```

Current revision and head match (`alembic current` and `alembic heads`):

```bash
docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env /app/venv/bin/alembic current 2>&1 | tail -1'
docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env /app/venv/bin/alembic heads 2>&1 | tail -1'
```
```
32f25cbf12f6 (head)
32f25cbf12f6 (head)
```

To prove the schema also builds cleanly **from scratch** — and that the mis-routing precondition is created by the migrations themselves — the full chain was applied to a throwaway database (`DB_URI=…/testmig`, dropped immediately afterward). It ran base → head; the final upgrade line was:

```bash
docker exec sl-app bash -lc 'cd /app && DB_URI=postgresql://test:test@localhost:15432/testmig CONFIG=tests/test.env /app/venv/bin/alembic upgrade head 2>&1' | grep "Running upgrade" | tail -1
```
```
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

**The schema fact at the heart of this diagnosis — `Contact.reply_email` is indexed but NOT unique.** The index definition carries no `UNIQUE`, and `pg_index.indisunique` is `f`:

```bash
docker exec sl-test-db psql -U test -d test -c "SELECT indexdef FROM pg_indexes WHERE indexname='ix_contact_reply_email';"
```
```
                                    indexdef
---------------------------------------------------------------------------------
 CREATE INDEX ix_contact_reply_email ON public.contact USING btree (reply_email)
(1 row)
```
```bash
docker exec sl-test-db psql -U test -d test -c "SELECT i.relname AS index, ix.indisunique FROM pg_class t JOIN pg_index ix ON t.oid=ix.indrelid JOIN pg_class i ON i.oid=ix.indexrelid WHERE t.relname='contact' AND i.relname='ix_contact_reply_email';"
```
```
         index          | indisunique
------------------------+-------------
 ix_contact_reply_email | f
(1 row)
```

`indisunique = f` confirms the index is non-unique, matching the migration source `migrations/versions/2021_071310_78403c7b8089_.py:22`:
```python
op.create_index(op.f('ix_contact_reply_email'), 'contact', ['reply_email'], unique=False)
```

**Database connection string used — `tests/test.env:17`:**

```bash
# tests/test.env:17
DB_URI=postgresql://test:test@localhost:15432/test
```
Verified live — the exact command (engine URL + `SELECT 1`) with its complete output:
```bash
docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env /app/venv/bin/python -c "from app.db import Session; from app.config import DB_URI; print(\"DB URL:\", DB_URI); print(\"connected:\", list(Session.execute(\"SELECT 1\"))[0][0])" 2>&1'
```
```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/fdewgwudahwiykesaxzf
Upload files to local dir
DB URL: postgresql://test:test@localhost:15432/test
connected: 1
```

> **DB-string reconciliation (stated per the task):** the optional helper `scripts/reset_test_db.sh:3` exports `DB_URI=postgresql://myuser:mypassword@localhost:15432/test` — same host/port/db (`15432/test`) but different credentials (`myuser:mypassword`). I used the `tests/test.env` string (`test:test`) because those are the credentials the running `sl-test-db` container was created with; the `reset_test_db.sh` credentials would fail authentication and were **not** used.

**DKIM key present (required for reply-delivery signing via `dkimpy`; `DKIM_PRIVATE_KEY_PATH=local_data/dkim.key` per `tests/test.env:15`).** The key is a **secret**, so only its presence, byte count, and SHA-256 digest are shown — the private-key material itself is never printed:

```bash
docker exec sl-app bash -lc 'test -s /app/local_data/dkim.key && echo "DKIM key file present: yes"; echo "byte count: $(wc -c < /app/local_data/dkim.key)"; echo "sha256: $(sha256sum /app/local_data/dkim.key | cut -d" " -f1)"'
```
```
DKIM key file present: yes
byte count: 886
sha256: 1c86748b8e445f96c4caaf1742a3d29fd628e5c08fc0dd9f0b45dd49a10326a6
```

The two drive commands (direct call and full SMTP socket) and the replay-loop command are given in sections (b)–(e) alongside their output.

---

## (b) Which component handles the inbound message

The inbound message is handled by the **aiosmtpd SMTP server** implemented in `email_handler.py`. The server is an `aiosmtpd` `Controller` wrapping a `MailHandler`; the entry chain is:

| Stage | Symbol | `file:line` |
|-------|--------|-------------|
| Server bootstrap | `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)` | `email_handler.py:2383` (in `main()` at `:2381`) |
| SMTP DATA hook | `async def handle_DATA(self, server, session, envelope)` | `email_handler.py:2289` |
| App-context wrapper | `def _handle(self, envelope, msg)` | `email_handler.py:2335` |
| Central router | `def handle(envelope, msg) -> str` | `email_handler.py:1945` |
| Reply-vs-forward dispatch | `if is_reverse_alias(rcpt_to):` → `handle_reply(...)` | `email_handler.py:2195` → `email_handler.py:2199` |
| Reply handler | `def handle_reply(envelope, msg, rcpt_to)` | `email_handler.py:966` |

Server bootstrap (verbatim):

```python
# email_handler.py:2381-2383
def main(port: int):
    """Use aiosmtpd Controller"""
    controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)
```

The dispatch that decides "this is a reply" (verbatim):

```python
# email_handler.py:2194-2200
        # Reply case: the recipient is a reverse alias. Used to start with "reply+" or "ra+"
        if is_reverse_alias(rcpt_to):
            LOG.d(
                "Reply phase %s(%s) -> %s", mail_from, copy_msg[headers.FROM], rcpt_to
            )
            is_delivered, smtp_status = handle_reply(envelope, copy_msg, rcpt_to)
            res.append((is_delivered, smtp_status))
```

### Evidence the message reached `handle_reply` via the real entry point

Two canonical drives were executed against the seeded fixtures (section (h) documents the fixture graph and the two-row precondition proof). Both delivered an SMTP message whose `RCPT TO` = the shared reverse-alias `re+shared@sl.local` and whose `MAIL FROM` = an authorized mailbox, and both reached `handle_reply()`.

**Drive 1 — direct call to the real function `email_handler.handle(envelope, msg)`** (this IS the production message-processing function and is the exact pattern used by `tests/test_email_handler.py` — e.g. `:274` — so it is canonical, not a bypass):

The script builds `envelope.mail_from = 'usera_obs@mailbox.test'` (Graph A's authorized mailbox), `envelope.rcpt_tos = ['re+shared@sl.local']`, and a minimal `EmailMessage` (`From`/`To`/`Subject`/body), then calls `email_handler.handle(envelope, msg)`.

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/drive_direct.py'
```

Complete, unedited output (the script's `print()`s interleaved with the handler's own `SL` log lines):

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/fgzqqztvuksxtfgpzifp
Upload files to local dir
>>> init logging <<<
2026-07-08 05:34:08,993 - SL - DEBUG - 14145 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
==== DRIVE 1 (direct canonical handle) ====
envelope.mail_from = usera_obs@mailbox.test
envelope.rcpt_tos  = ['re+shared@sl.local']
2026-07-08 05:34:09,736 - SL - INFO - 14145 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:34:09,737 - SL - DEBUG - 14145 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 05:34:09,891 - SL - DEBUG - 14145 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:usera_obs@mailbox.test, rcpt_tos:['re+shared@sl.local'], header_from:usera_obs@mailbox.test, header_to:re+shared@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'usera_obs@mailbox.test'), ('To', 're+shared@sl.local'), ('Subject', 'obs reply drive1'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 05:34:09,897 - SL - DEBUG - 14145 - "/app/email_handler.py:2196" - handle() -  - Reply phase usera_obs@mailbox.test(usera_obs@mailbox.test) -> re+shared@sl.local
2026-07-08 05:34:09,912 - SL - INFO - 14145 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:34:09,916 - SL - DEBUG - 14145 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 559> for <Contact 212 alice_real@external-A.test 1341>, <User 829 User A Obs usera_obs@mailbox.test>, <Mailbox 981 usera_obs@mailbox.test>
2026-07-08 05:34:09,922 - SL - DEBUG - 14145 - "/app/email_handler.py:1120" - handle_reply() -  - Replace reverse-alias re+shared@sl.local by contact email <Contact 212 alice_real@external-A.test 1341>
2026-07-08 05:34:09,923 - SL - DEBUG - 14145 - "/app/email_handler.py:1122" - handle_reply() -  - Replace mailbox usera_obs@mailbox.test by alias email booked_reeves953@sl.local
2026-07-08 05:34:09,926 - SL - DEBUG - 14145 - "/app/email_handler.py:1141" - handle_reply() -  - Replace reverse alias by real address for 1 contacts takes 0.0009496212005615234 seconds
2026-07-08 05:34:09,928 - SL - DEBUG - 14145 - "/app/email_handler.py:1171" - handle_reply() -  - From header is booked_reeves953@sl.local
2026-07-08 05:34:09,930 - SL - DEBUG - 14145 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: re+shared@sl.local, new: alice_real@external-A.test
2026-07-08 05:34:09,930 - SL - DEBUG - 14145 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-08 05:34:09,931 - SL - DEBUG - 14145 - "/app/email_handler.py:1336" - replace_original_message_id() -  - no original_message_id, create a new sl_message_id <178348884993.14145.6230102855616134613.559@sl.local>
2026-07-08 05:34:09,937 - SL - WARNING - 14145 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 05:34:09,940 - SL - DEBUG - 14145 - "/app/email_handler.py:1212" - handle_reply() -  - send email from booked_reeves953@sl.local to alice_real@external-A.test, mail_options:[],rcpt_options:[]
2026-07-08 05:34:09,944 - SL - DEBUG - 14145 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs reply drive1', from 'booked_reeves953@sl.local' to 'alice_real@external-A.test'
handle() return status = '250 Message accepted for delivery'
stored email count = 1
  [stored 0] envelope_from='sl.lmysyibvgu4syibsgm3tiobzgroq.uukq6ciwe3yg2@sl.local' envelope_to='alice_real@external-A.test' is_forward=False
---- EmailLog written ----
  EmailLog.id=559 user_id=829 alias_id=1341 mailbox_id=981 contact_id=212
  contact.website_email (delivery recipient) = 'alice_real@external-A.test'
  => resolved to GraphA/Alice(user 829)
```

The dispatch log line at `email_handler.py:2196` — `Reply phase usera_obs@mailbox.test(usera_obs@mailbox.test) -> re+shared@sl.local`, shown complete in the Drive 2 trace above — is emitted because `is_reverse_alias(rcpt_to)` at `email_handler.py:2195` returned `True` (**the alias is recognized**), and control then entered `handle_reply()` at `email_handler.py:2199`. The returned SMTP status was `'250 Message accepted for delivery'` (`status.E200`, `app/email/status.py:2`).

**Drive 2 — full aiosmtpd SMTP socket** (highest fidelity: real `Controller(MailHandler())` listening on a socket, exactly as `email_handler.main()` builds it at `:2383`):

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/drive_smtp.py'
```

Complete, unedited output — the SMTP dialogue (`[EHLO]`/`[MAIL FROM]`/`[RCPT TO]`/`[DATA]`) and the full handler chain via the socket (note the per-message id `8482a677-…` threaded through every log line):

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ntebmkqbnxyanxhyexqi
Upload files to local dir
>>> init logging <<<
2026-07-08 05:34:57,733 - SL - DEBUG - 14173 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
==== DRIVE 2 (real aiosmtpd Controller socket) ====
Controller started 127.0.0.1:20399  handler='MailHandler'
[connect] banner: True
[EHLO] code=250 msg=b'reverse-code-generator-761c1c6d-7x8h9\nSIZE 33554432\n8BITMIME\nSMTPUTF8\nHELP'
[MAIL FROM] code=250 msg=b'OK'
[RCPT TO] code=250 msg=b'OK'
2026-07-08 05:34:58,530 - SL - DEBUG - 14173 - "/app/app/log.py:24" - set_message_id() -  - set message_id 8482a677-656a-4426-a492-c3c1248499e5
2026-07-08 05:34:58,531 - SL - DEBUG - 14173 - "/app/email_handler.py:2342" - _handle() - 8482a677-656a-4426-a492-c3c1248499e5 - ====>=====>====>====>====>====>====>====>
2026-07-08 05:34:58,531 - SL - INFO - 14173 - "/app/email_handler.py:2343" - _handle() - 8482a677-656a-4426-a492-c3c1248499e5 - New message, mail from usera_obs@mailbox.test, rctp tos ['re+shared@sl.local'] 
2026-07-08 05:34:58,532 - SL - INFO - 14173 - "/app/email_handler.py:1956" - handle() - 8482a677-656a-4426-a492-c3c1248499e5 - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:34:58,532 - SL - DEBUG - 14173 - "/app/email_handler.py:1963" - handle() - 8482a677-656a-4426-a492-c3c1248499e5 - Cannot parse Postfix queue ID from None None
2026-07-08 05:34:58,699 - SL - DEBUG - 14173 - "/app/email_handler.py:1980" - handle() - 8482a677-656a-4426-a492-c3c1248499e5 - ==>> Handle mail_from:usera_obs@mailbox.test, rcpt_tos:['re+shared@sl.local'], header_from:usera_obs@mailbox.test, header_to:re+shared@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'usera_obs@mailbox.test'), ('To', 're+shared@sl.local'), ('Subject', 'obs reply drive2 real socket'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 05:34:58,705 - SL - DEBUG - 14173 - "/app/email_handler.py:2196" - handle() - 8482a677-656a-4426-a492-c3c1248499e5 - Reply phase usera_obs@mailbox.test(usera_obs@mailbox.test) -> re+shared@sl.local
2026-07-08 05:34:58,722 - SL - INFO - 14173 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 8482a677-656a-4426-a492-c3c1248499e5 - DMARC check disabled
2026-07-08 05:34:58,730 - SL - DEBUG - 14173 - "/app/email_handler.py:1051" - handle_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - Create <EmailLog 560> for <Contact 212 alice_real@external-A.test 1341>, <User 829 User A Obs usera_obs@mailbox.test>, <Mailbox 981 usera_obs@mailbox.test>
2026-07-08 05:34:58,735 - SL - DEBUG - 14173 - "/app/email_handler.py:1120" - handle_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - Replace reverse-alias re+shared@sl.local by contact email <Contact 212 alice_real@external-A.test 1341>
2026-07-08 05:34:58,737 - SL - DEBUG - 14173 - "/app/email_handler.py:1122" - handle_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - Replace mailbox usera_obs@mailbox.test by alias email booked_reeves953@sl.local
2026-07-08 05:34:58,740 - SL - DEBUG - 14173 - "/app/email_handler.py:1141" - handle_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - Replace reverse alias by real address for 1 contacts takes 0.0008389949798583984 seconds
2026-07-08 05:34:58,742 - SL - DEBUG - 14173 - "/app/email_handler.py:1171" - handle_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - From header is booked_reeves953@sl.local
2026-07-08 05:34:58,743 - SL - DEBUG - 14173 - "/app/email_handler.py:380" - replace_header_when_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - Replace To header, old: re+shared@sl.local, new: alice_real@external-A.test
2026-07-08 05:34:58,743 - SL - DEBUG - 14173 - "/app/email_handler.py:383" - replace_header_when_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - delete the Cc header. Old value None
2026-07-08 05:34:58,744 - SL - DEBUG - 14173 - "/app/email_handler.py:1336" - replace_original_message_id() - 8482a677-656a-4426-a492-c3c1248499e5 - no original_message_id, create a new sl_message_id <178348889874.14173.7992613772673903151.560@sl.local>
2026-07-08 05:34:58,746 - SL - WARNING - 14173 - "/app/email_handler.py:1206" - handle_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - missing date header, add one
2026-07-08 05:34:58,749 - SL - DEBUG - 14173 - "/app/email_handler.py:1212" - handle_reply() - 8482a677-656a-4426-a492-c3c1248499e5 - send email from booked_reeves953@sl.local to alice_real@external-A.test, mail_options:[],rcpt_options:[]
2026-07-08 05:34:58,752 - SL - DEBUG - 14173 - "/app/app/mail_sender.py:131" - send() - 8482a677-656a-4426-a492-c3c1248499e5 - send email with subject 'obs reply drive2 real socket', from 'booked_reeves953@sl.local' to 'alice_real@external-A.test'
2026-07-08 05:34:58,755 - SL - INFO - 14173 - "/app/email_handler.py:2367" - _handle() - 8482a677-656a-4426-a492-c3c1248499e5 - Finish mail_from usera_obs@mailbox.test, rcpt_tos ['re+shared@sl.local'], takes 0.22440314292907715 seconds with return code '250 Message accepted for delivery'<<===
[DATA] code=250 msg=b'Message accepted for delivery'
---- newest EmailLog after socket drive ----
  EmailLog.id=560 user_id=829 alias_id=1341 mailbox_id=981 contact_id=212
  delivery recipient contact.website_email = 'alice_real@external-A.test'
  => resolved to GraphA/Alice(user 829)
```

Both drives traversed `handle_DATA` (`:2289`) → `_handle` (`:2335`) → `handle` (`:1945`) → dispatch (`:2195`/`:2199`) → `handle_reply` (`:966`) — visible in Drive 2 as `_handle()@2342-2343 → handle()@1956,1980 → @2196 (dispatch) → handle_reply()@1051 → _handle()@2367 (finish)` — confirming the **aiosmtpd server in `email_handler.py` is the component that handles the incoming message**.


---

## (c) How the alias is resolved to a user

Inside `handle_reply()` the resolution is **single-sourced**: one `Contact` row is fetched, and the alias, user, and authorizing mailbox are all derived from it. The verbatim chain:

```python
# email_handler.py:984-1004 (excerpt)
    # handle case where reply email is generated with non-allowed char
    reply_email = normalize_reply_email(reply_email)      # :984

    contact = Contact.get_by(reply_email=reply_email)      # :986  <-- single-sourced resolution (PRIME suspect)
    if not contact:
        LOG.w(f"No contact with {reply_email} as reverse alias")
        return False, status.E502                          # :989
    if not contact.user.is_active():
        LOG.w(f"User {contact.user} has been soft deleted")
        return False, status.E502                          # :992

    alias = contact.alias                                  # :994
    alias_address: str = contact.alias.email
    alias_domain = get_email_domain_part(alias_address)

    # Sanity check: verify alias domain is managed by SimpleLogin
    if not is_valid_alias_address_domain(alias.email):
        LOG.e("%s domain isn't known", alias)
        return False, status.E503                          # :1002

    user = alias.user                                      # :1004
```

Resolution steps, by name and line:

1. **Normalize the lookup key** — `reply_email = normalize_reply_email(reply_email)` at **`email_handler.py:984`** (defined `app/email_validation.py:25`). See section (f) for how this *widens* the collision surface.
2. **Fetch the single contact** — `contact = Contact.get_by(reply_email=reply_email)` at **`email_handler.py:986`**. This is the `ModelMixin.get_by` helper (`app/models.py:83-84`) → `filter_by(reply_email=...).first()` with **no `ORDER BY`**.
3. **Derive the alias from the contact** — `alias = contact.alias` at **`email_handler.py:994`**. The alias is *not* looked up independently; it is read off the resolved contact.
4. **Derive the user from the alias** — `user = alias.user` at **`email_handler.py:1004`**.
5. **Authorize via the mailbox** — `mailbox = get_mailbox_from_mail_from(mail_from, alias)` at **`email_handler.py:1019`** (defined `email_handler.py:1364`), which checks the `MAIL FROM` against `alias.mailboxes`.

The dispatch's recognition step uses the **same** helper on the same column:

```python
# app/email_utils.py:1156-1163
def is_reverse_alias(address: str) -> bool:
    # to take into account the new reverse-alias that doesn't start with "ra+"
    if Contact.get_by(reply_email=address):
        return True

    return address.endswith(f"@{config.EMAIL_DOMAIN}") and (
        address.startswith("reply+") or address.startswith("ra+")
    )
```

**Key insight (why "the alias is recognized" yet the user can be wrong):** `is_reverse_alias()` at `app/email_utils.py:1158` returns `True` as soon as *any* `Contact` with that `reply_email` **exists** — it does not care *which* row. The **identity** is then resolved *separately* at `email_handler.py:986` by another unordered `.first()`. So recognition (logged at `email_handler.py:2196`) succeeds on the existence of either duplicate, while the identity chosen at `:986` may be the *other* row.

### Captured evidence of the resolution chain (BEFORE / DURING)

The chain was instrumented **externally** from an ephemeral `/tmp/observe_chain.py` script that (a) printed the **compiled SQL** for `Contact.get_by(reply_email=...)`, (b) listed the candidate rows **before** the lookup, (c) monkeypatched `Contact.get_by` **at runtime** to log every call and its returned row **during** the real `handle()`, and (d) read back the committed `EmailLog` **after**. The source file was never edited; the monkeypatch lives only in the ephemeral process and is discarded at exit. Exact command:

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/observe_chain.py'
```

Complete, unedited output for the BEFORE and DURING stages (the compiled SQL is reproduced in section (e); the AFTER stage is in section (d) below — all four stages are one continuous run of this script):

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ezgruqljwnteqkksotsy
Upload files to local dir
>>> init logging <<<
2026-07-08 05:35:58,321 - SL - DEBUG - 14202 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt

==== BEFORE lookup ====
reply_email key = 're+shared@sl.local'
rows matching this reply_email = 2
  candidate Contact id=212 alias_id=1341 user_id=829 website_email='alice_real@external-A.test'
  candidate Contact id=213 alias_id=1343 user_id=830 website_email='bob_real@external-B.test'

==== DURING: real handle() executes ====
2026-07-08 05:35:59,184 - SL - INFO - 14202 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:35:59,184 - SL - DEBUG - 14202 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 05:35:59,186 - SL - DEBUG - 14202 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:usera_obs@mailbox.test, rcpt_tos:['re+shared@sl.local'], header_from:usera_obs@mailbox.test, header_to:re+shared@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'usera_obs@mailbox.test'), ('To', 're+shared@sl.local'), ('Subject', 'obs chain'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
[DURING Contact.get_by] kw={'reply_email': 'usera_obs@mailbox.test'} -> Contact id=None alias_id=None user_id=None
[DURING Contact.get_by] kw={'reply_email': 'usera_obs@mailbox.test'} -> Contact id=None alias_id=None user_id=None
[DURING Contact.get_by] kw={'reply_email': 're+shared@sl.local'} -> Contact id=212 alias_id=1341 user_id=829
[DURING Contact.get_by] kw={'reply_email': 're+shared@sl.local'} -> Contact id=212 alias_id=1341 user_id=829
2026-07-08 05:35:59,190 - SL - DEBUG - 14202 - "/app/email_handler.py:2196" - handle() -  - Reply phase usera_obs@mailbox.test(usera_obs@mailbox.test) -> re+shared@sl.local
[DURING Contact.get_by] kw={'reply_email': 're+shared@sl.local'} -> Contact id=212 alias_id=1341 user_id=829
2026-07-08 05:35:59,205 - SL - INFO - 14202 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:35:59,209 - SL - DEBUG - 14202 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 561> for <Contact 212 alice_real@external-A.test 1341>, <User 829 User A Obs usera_obs@mailbox.test>, <Mailbox 981 usera_obs@mailbox.test>
2026-07-08 05:35:59,215 - SL - DEBUG - 14202 - "/app/email_handler.py:1120" - handle_reply() -  - Replace reverse-alias re+shared@sl.local by contact email <Contact 212 alice_real@external-A.test 1341>
2026-07-08 05:35:59,216 - SL - DEBUG - 14202 - "/app/email_handler.py:1122" - handle_reply() -  - Replace mailbox usera_obs@mailbox.test by alias email booked_reeves953@sl.local
2026-07-08 05:35:59,219 - SL - DEBUG - 14202 - "/app/email_handler.py:1141" - handle_reply() -  - Replace reverse alias by real address for 1 contacts takes 0.0008378028869628906 seconds
2026-07-08 05:35:59,221 - SL - DEBUG - 14202 - "/app/email_handler.py:1171" - handle_reply() -  - From header is booked_reeves953@sl.local
[DURING Contact.get_by] kw={'reply_email': 're+shared@sl.local'} -> Contact id=212 alias_id=1341 user_id=829
2026-07-08 05:35:59,222 - SL - DEBUG - 14202 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: re+shared@sl.local, new: alice_real@external-A.test
2026-07-08 05:35:59,223 - SL - DEBUG - 14202 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-08 05:35:59,223 - SL - DEBUG - 14202 - "/app/email_handler.py:1336" - replace_original_message_id() -  - no original_message_id, create a new sl_message_id <178348895922.14202.16380086965538075397.561@sl.local>
2026-07-08 05:35:59,231 - SL - WARNING - 14202 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 05:35:59,234 - SL - DEBUG - 14202 - "/app/email_handler.py:1212" - handle_reply() -  - send email from booked_reeves953@sl.local to alice_real@external-A.test, mail_options:[],rcpt_options:[]
2026-07-08 05:35:59,237 - SL - DEBUG - 14202 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs chain', from 'booked_reeves953@sl.local' to 'alice_real@external-A.test'
handle() ->  '250 Message accepted for delivery'
```

Reading the DURING trace: `Contact.get_by` is invoked **six** times in this single message. The first two calls carry `reply_email='usera_obs@mailbox.test'` (the `MAIL FROM`, checked while classifying the sender) and return `None`; the remaining four carry the reverse-alias key `reply_email='re+shared@sl.local'` — the recognition call behind `is_reverse_alias` (dispatch, logged at `email_handler.py:2196`), the identity call at `handle_reply:986`, and two later header-rewrite reads — and **every one returned `Contact id=212 … user_id=829`** in this run. Both the recognition call and the identity call hit the same unordered `.first()`; here they agreed on Contact 212, but section (e) shows that agreement is not guaranteed across DB/plan states.


---

## (d) What `user_id` the system decides to forward the reply to

The routing decision is committed to an `EmailLog` row, and the reply is delivered to the contact's real address. Both are taken **directly from the single resolved `Contact` row**. Verbatim:

```python
# email_handler.py:1042-1050 (the EmailLog.create call spans lines 1042-1050;
#                             the LOG.d below is a separate statement at :1051)
    email_log = EmailLog.create(                          # :1042 opens the call
        contact_id=contact.id,
        alias_id=contact.alias_id,
        is_reply=True,
        user_id=contact.user_id,
        mailbox_id=mailbox.id,
        message_id=msg[headers.MESSAGE_ID],
        commit=True,
    )                                                     # :1050 closes the call
    LOG.d("Create %s for %s, %s, %s", email_log, contact, user, mailbox)   # :1051
```

```python
# email_handler.py:1119-1121
    if user.replace_reverse_alias:
        LOG.d("Replace reverse-alias %s by contact email %s", reply_email, contact)
        msg = replace(msg, reply_email, contact.website_email)
```

- The **chosen `user_id`** is `contact.user_id` at **`email_handler.py:1046`**.
- The **authorizing mailbox** is `mailbox.id` at **`email_handler.py:1047`**.
- The **delivery recipient** is `contact.website_email` at **`email_handler.py:1121`**.

### Captured evidence (AFTER) — the fully-worked single-message trace

From the same `/tmp/observe_chain.py` run as section (c) (the AFTER stage of the identical command), the committed decision and delivery, complete and unedited:

```
==== AFTER routing decision committed ====
  EmailLog.id=561 user_id=829 alias_id=1341 mailbox_id=981 contact_id=212
  alias.user_id (derived) = 829
  delivery recipient contact.website_email = 'alice_real@external-A.test'
  stored envelope_to = ['alice_real@external-A.test']
```

The DURING trace above (section (c)) showed `handle() ->  '250 Message accepted for delivery'` (`status.E200`); this AFTER block reads the row that decision committed: `EmailLog 561`, whose `user_id=829` and `mailbox_id=981` are exactly Graph A's identifiers.

The actual outbound recipient was observed (not merely read from code) via the mail-capture hook `app.mail_sender.get_stored_emails()` / `store_emails_test_decorator` (`app/mail_sender.py:108`, `:111`); the captured `SendRequest.envelope_to` is the real recipient the reply was relayed to.

### Proof the entire routing is single-sourced from the `Contact` row at `email_handler.py:986`

For the single worked message, every downstream identifier traces to the one row returned at `:986`:

| Value | Source expression | `file:line` | Observed |
|-------|-------------------|-------------|----------|
| resolved contact | `Contact.get_by(reply_email=...)` | `email_handler.py:986` | `contact.id = 212` |
| alias | `alias = contact.alias` | `email_handler.py:994` | `alias.id = 1341` |
| user | `user = alias.user` | `email_handler.py:1004` | `user.id = 829` |
| logged `user_id` | `EmailLog.create(user_id=contact.user_id)` | `email_handler.py:1046` | `829` |
| mailbox | `mailbox.id` (via `get_mailbox_from_mail_from`) | `email_handler.py:1047` | `981` (Alias A's mailbox) |
| recipient | `replace(msg, reply_email, contact.website_email)` | `email_handler.py:1121` | `alice_real@external-A.test` |

**Conclusion:** the `user_id` the system forwards to is whatever `user_id` belongs to the single row returned at `email_handler.py:986`. In this run that was `829` (Graph A). Had `:986` returned Contact 213 instead, the logged `user_id` would be `830` and the recipient `bob_real@external-B.test` — **the wrong user** — which is exactly what section (e) reproduces.


---

## (e) The observed distribution of resolved `user_id` across repeated identical runs

**The mechanism.** `ModelMixin.get_by` emits a `filter_by(...).first()` with **no `ORDER BY`**:

```python
# app/models.py:83-84
    def get_by(cls, **kw):
        return Session.query(cls).filter_by(**kw).first()
```

The exact SQL this compiles to was captured (complete, unedited) by the first stage of `/tmp/observe_chain.py` (the same script used in section (c)); it prints the SQLAlchemy 1.3.24 compilation of `Contact.get_by(reply_email='re+shared@sl.local')` for both `filter_by(...)` and `filter_by(...).first()`:

```
==== COMPILED SQL for Contact.get_by(reply_email=...) ====
filter_by(...) statement:
SELECT contact.id, contact.created_at, contact.updated_at, contact.user_id, contact.alias_id, contact.name, contact.website_email, contact.website_from, contact.reply_email, contact.is_cc, contact.pgp_public_key, contact.pgp_finger_print, contact.mail_from, contact.invalid_email, contact.block_forward, contact.automatic_created, contact.flags 
FROM contact 
WHERE contact.reply_email = 're+shared@sl.local'

.first() appends LIMIT 1 (SQLAlchemy). filter_by(...).limit(1) statement:
SELECT contact.id, contact.created_at, contact.updated_at, contact.user_id, contact.alias_id, contact.name, contact.website_email, contact.website_from, contact.reply_email, contact.is_cc, contact.pgp_public_key, contact.pgp_finger_print, contact.mail_from, contact.invalid_email, contact.block_forward, contact.automatic_created, contact.flags 
FROM contact 
WHERE contact.reply_email = 're+shared@sl.local'
 LIMIT 1
```

`.first()` appends ` LIMIT 1` and there is **no `ORDER BY`** in the emitted statement — so the row returned among the two duplicates is whichever the query plan/physical order yields; the application controls neither.

**Replay-loop harness.** `/tmp/repro_loop.py` takes two arguments — an iteration count and a run label — and replays the **identical** inbound reply through the real `email_handler.handle()` entry that many times (same `Envelope` `mail_from=usera_obs@mailbox.test`, `rcpt_tos=['re+shared@sl.local']`, same `EmailMessage`); every invocation below uses a count of `30`. After each call it reads back the just-created `EmailLog` row (`EmailLog.filter(contact_id.in_([212,213])).order_by(id.desc()).first()`) to tabulate the resolved `contact_id`/`user_id`. The input never changes between iterations; only the database's physical/plan state — manipulated externally by `psql` between runs — differs. Nothing in the repository is modified. The demonstration below is one continuous, reversible sequence: **baseline → RUN1 → flip → RUN2 → RUN2b → EXPLAIN both plans → flip-back → RUN3.**

### Baseline physical order and unordered `LIMIT 1` winner

```bash
docker exec -i sl-test-db psql -U test -d test < /tmp/obs_build/e_show.sql
# e_show.sql:
#   SELECT ctid, id, user_id FROM contact WHERE reply_email='re+shared@sl.local' ORDER BY ctid;
#   SELECT id, user_id FROM contact WHERE reply_email='re+shared@sl.local' LIMIT 1;
```
```
=== rows sharing re+shared@sl.local, in physical (ctid) order ===
  ctid  | id  | user_id 
--------+-----+---------
 (1,15) | 212 |     829
 (1,16) | 213 |     830
(2 rows)

=== what the unordered "... LIMIT 1" returns ===
 id  | user_id 
-----+---------
 212 |     829
(1 row)
```
→ at baseline, Contact **212** (user **829**, Alice) is physically first at `ctid (1,15)`, so the unordered `... LIMIT 1` returns it.

### Run 1 — replay the identical input N=30 at baseline order

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/repro_loop.py 30 RUN1_before_flip'
```
```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/yqvgawkimahghcwwryly
Upload files to local dir
>>> init logging <<<
2026-07-08 06:03:58,619 - SL - DEBUG - 14685 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
======== DISTRIBUTION [RUN1_before_flip] N=30 ========
  run  0 -> contact_id=212 user_id=829 [Alice(A)]
  run  1 -> contact_id=212 user_id=829 [Alice(A)]
  run  2 -> contact_id=212 user_id=829 [Alice(A)]
  run  3 -> contact_id=212 user_id=829 [Alice(A)]
  run  4 -> contact_id=212 user_id=829 [Alice(A)]
  run  5 -> contact_id=212 user_id=829 [Alice(A)]
  run  6 -> contact_id=212 user_id=829 [Alice(A)]
  run  7 -> contact_id=212 user_id=829 [Alice(A)]
  run  8 -> contact_id=212 user_id=829 [Alice(A)]
  run  9 -> contact_id=212 user_id=829 [Alice(A)]
  run 10 -> contact_id=212 user_id=829 [Alice(A)]
  run 11 -> contact_id=212 user_id=829 [Alice(A)]
  run 12 -> contact_id=212 user_id=829 [Alice(A)]
  run 13 -> contact_id=212 user_id=829 [Alice(A)]
  run 14 -> contact_id=212 user_id=829 [Alice(A)]
  run 15 -> contact_id=212 user_id=829 [Alice(A)]
  run 16 -> contact_id=212 user_id=829 [Alice(A)]
  run 17 -> contact_id=212 user_id=829 [Alice(A)]
  run 18 -> contact_id=212 user_id=829 [Alice(A)]
  run 19 -> contact_id=212 user_id=829 [Alice(A)]
  run 20 -> contact_id=212 user_id=829 [Alice(A)]
  run 21 -> contact_id=212 user_id=829 [Alice(A)]
  run 22 -> contact_id=212 user_id=829 [Alice(A)]
  run 23 -> contact_id=212 user_id=829 [Alice(A)]
  run 24 -> contact_id=212 user_id=829 [Alice(A)]
  run 25 -> contact_id=212 user_id=829 [Alice(A)]
  run 26 -> contact_id=212 user_id=829 [Alice(A)]
  run 27 -> contact_id=212 user_id=829 [Alice(A)]
  run 28 -> contact_id=212 user_id=829 [Alice(A)]
  run 29 -> contact_id=212 user_id=829 [Alice(A)]
  --- summary ---
  contact_id distribution: {212: 30}
  user_id    distribution: {829: 30}
  => contact 212 (user 829, Alice) = 30/30 ; contact 213 (user 830, Bob) = 0/30
```
→ 100% **user 829** (Graph A / Alice): 30/30. The identical input is **process-stable** at this DB state.

### Flip the physical order with a benign write, then replay the identical input

A routine write to Contact 212 (the kind any ordinary contact update performs — an `updated_at` bump) relocates its row under MVCC to a new heap slot, changing which duplicate is physically first:

```bash
docker exec -i sl-test-db psql -U test -d test < /tmp/obs_build/e_flip.sql
# e_flip.sql:
#   UPDATE contact SET updated_at = now() WHERE id = 212;
#   SELECT ctid, id, user_id FROM contact WHERE reply_email='re+shared@sl.local' ORDER BY ctid;
#   SELECT id, user_id FROM contact WHERE reply_email='re+shared@sl.local' LIMIT 1;
```
```
=== FLIP: UPDATE contact 212 (touch updated_at) -> relocates its MVCC tuple to a new heap slot ===
UPDATE 1
=== ctid AFTER flip (physical scan order) ===
  ctid  | id  | user_id 
--------+-----+---------
 (1,16) | 213 |     830
 (1,17) | 212 |     829
(2 rows)

=== what the unordered "... LIMIT 1" returns now ===
 id  | user_id 
-----+---------
 213 |     830
(1 row)
```
The `UPDATE` moved Contact 212 from `ctid (1,15)` to `(1,17)`, so Contact **213** (user **830**, Bob) is now physically first at `(1,16)`; the unordered `... LIMIT 1` now returns **213 / user 830** — with **no change whatsoever to the inbound message**.

### Run 2 — identical input, after the benign write

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/repro_loop.py 30 RUN2_after_flip'
```
```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/cgsayfxnhdbsoxphqywi
Upload files to local dir
>>> init logging <<<
2026-07-08 06:04:01,864 - SL - DEBUG - 14704 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
======== DISTRIBUTION [RUN2_after_flip] N=30 ========
  run  0 -> contact_id=213 user_id=830 [Bob(B)]
  run  1 -> contact_id=213 user_id=830 [Bob(B)]
  run  2 -> contact_id=213 user_id=830 [Bob(B)]
  run  3 -> contact_id=213 user_id=830 [Bob(B)]
  run  4 -> contact_id=213 user_id=830 [Bob(B)]
  run  5 -> contact_id=213 user_id=830 [Bob(B)]
  run  6 -> contact_id=213 user_id=830 [Bob(B)]
  run  7 -> contact_id=213 user_id=830 [Bob(B)]
  run  8 -> contact_id=213 user_id=830 [Bob(B)]
  run  9 -> contact_id=213 user_id=830 [Bob(B)]
  run 10 -> contact_id=213 user_id=830 [Bob(B)]
  run 11 -> contact_id=213 user_id=830 [Bob(B)]
  run 12 -> contact_id=213 user_id=830 [Bob(B)]
  run 13 -> contact_id=213 user_id=830 [Bob(B)]
  run 14 -> contact_id=213 user_id=830 [Bob(B)]
  run 15 -> contact_id=213 user_id=830 [Bob(B)]
  run 16 -> contact_id=213 user_id=830 [Bob(B)]
  run 17 -> contact_id=213 user_id=830 [Bob(B)]
  run 18 -> contact_id=213 user_id=830 [Bob(B)]
  run 19 -> contact_id=213 user_id=830 [Bob(B)]
  run 20 -> contact_id=213 user_id=830 [Bob(B)]
  run 21 -> contact_id=213 user_id=830 [Bob(B)]
  run 22 -> contact_id=213 user_id=830 [Bob(B)]
  run 23 -> contact_id=213 user_id=830 [Bob(B)]
  run 24 -> contact_id=213 user_id=830 [Bob(B)]
  run 25 -> contact_id=213 user_id=830 [Bob(B)]
  run 26 -> contact_id=213 user_id=830 [Bob(B)]
  run 27 -> contact_id=213 user_id=830 [Bob(B)]
  run 28 -> contact_id=213 user_id=830 [Bob(B)]
  run 29 -> contact_id=213 user_id=830 [Bob(B)]
  --- summary ---
  contact_id distribution: {213: 30}
  user_id    distribution: {830: 30}
  => contact 212 (user 829, Alice) = 0/30 ; contact 213 (user 830, Bob) = 30/30
```
→ 100% **user 830 = THE WRONG USER**: 30/30 to Bob. **Same unchanged input, same recognized alias, different user.** This is a live reproduction of the reported symptom.

### Run 2b — stability confirmation (≥2 runs, same state)

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/repro_loop.py 30 RUN2b_after_flip'
```
```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/kjumqnljpwztceisxdaf
Upload files to local dir
>>> init logging <<<
2026-07-08 06:04:05,033 - SL - DEBUG - 14723 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
======== DISTRIBUTION [RUN2b_after_flip] N=30 ========
  run  0 -> contact_id=213 user_id=830 [Bob(B)]
  run  1 -> contact_id=213 user_id=830 [Bob(B)]
  run  2 -> contact_id=213 user_id=830 [Bob(B)]
  run  3 -> contact_id=213 user_id=830 [Bob(B)]
  run  4 -> contact_id=213 user_id=830 [Bob(B)]
  run  5 -> contact_id=213 user_id=830 [Bob(B)]
  run  6 -> contact_id=213 user_id=830 [Bob(B)]
  run  7 -> contact_id=213 user_id=830 [Bob(B)]
  run  8 -> contact_id=213 user_id=830 [Bob(B)]
  run  9 -> contact_id=213 user_id=830 [Bob(B)]
  run 10 -> contact_id=213 user_id=830 [Bob(B)]
  run 11 -> contact_id=213 user_id=830 [Bob(B)]
  run 12 -> contact_id=213 user_id=830 [Bob(B)]
  run 13 -> contact_id=213 user_id=830 [Bob(B)]
  run 14 -> contact_id=213 user_id=830 [Bob(B)]
  run 15 -> contact_id=213 user_id=830 [Bob(B)]
  run 16 -> contact_id=213 user_id=830 [Bob(B)]
  run 17 -> contact_id=213 user_id=830 [Bob(B)]
  run 18 -> contact_id=213 user_id=830 [Bob(B)]
  run 19 -> contact_id=213 user_id=830 [Bob(B)]
  run 20 -> contact_id=213 user_id=830 [Bob(B)]
  run 21 -> contact_id=213 user_id=830 [Bob(B)]
  run 22 -> contact_id=213 user_id=830 [Bob(B)]
  run 23 -> contact_id=213 user_id=830 [Bob(B)]
  run 24 -> contact_id=213 user_id=830 [Bob(B)]
  run 25 -> contact_id=213 user_id=830 [Bob(B)]
  run 26 -> contact_id=213 user_id=830 [Bob(B)]
  run 27 -> contact_id=213 user_id=830 [Bob(B)]
  run 28 -> contact_id=213 user_id=830 [Bob(B)]
  run 29 -> contact_id=213 user_id=830 [Bob(B)]
  --- summary ---
  contact_id distribution: {213: 30}
  user_id    distribution: {830: 30}
  => contact 212 (user 829, Alice) = 0/30 ; contact 213 (user 830, Bob) = 30/30
```
→ characterization **stable across a second, independent run** (RUN2b = 30/30 Bob, identical to RUN2): within this fixed DB state the outcome is 100% deterministic.

### Query-plan dimension (same flipped DB state as Run 2)

The choice of query plan alone selects a different duplicate, with **no data change at all** — at the flipped state, the same unordered `... LIMIT 1` returns Bob under the default sequential scan but Alice under a forced index scan:

```bash
docker exec -i sl-test-db psql -U test -d test < /tmp/obs_build/e_explain.sql
# e_explain.sql:
#   EXPLAIN (ANALYZE, COSTS OFF) SELECT contact.id FROM contact WHERE contact.reply_email = 're+shared@sl.local' LIMIT 1;
#   SELECT id, user_id FROM contact WHERE reply_email='re+shared@sl.local' LIMIT 1;
#   SET enable_seqscan = off;
#   EXPLAIN (ANALYZE, COSTS OFF) SELECT contact.id FROM contact WHERE contact.reply_email = 're+shared@sl.local' LIMIT 1;
#   SELECT id, user_id FROM contact WHERE reply_email='re+shared@sl.local' LIMIT 1;
#   RESET enable_seqscan;
```
```
=== PLAN 1: default plan -- EXPLAIN (ANALYZE, COSTS OFF), same unordered query ===
                             QUERY PLAN                              
---------------------------------------------------------------------
 Limit (actual time=0.029..0.030 rows=1 loops=1)
   ->  Seq Scan on contact (actual time=0.028..0.028 rows=1 loops=1)
         Filter: ((reply_email)::text = 're+shared@sl.local'::text)
         Rows Removed by Filter: 44
 Planning Time: 0.482 ms
 Execution Time: 0.049 ms
(6 rows)

 id  | user_id 
-----+---------
 213 |     830
(1 row)

=== PLAN 2: forced index scan (SET enable_seqscan=off) -- SAME query, no ORDER BY ===
SET
                                             QUERY PLAN                                             
----------------------------------------------------------------------------------------------------
 Limit (actual time=0.007..0.008 rows=1 loops=1)
   ->  Index Scan using ix_contact_reply_email on contact (actual time=0.007..0.007 rows=1 loops=1)
         Index Cond: ((reply_email)::text = 're+shared@sl.local'::text)
 Planning Time: 0.025 ms
 Execution Time: 0.014 ms
(5 rows)

 id  | user_id 
-----+---------
 212 |     829
(1 row)

RESET
```
→ the **two valid plans** for the **identical, unordered** query return **different users**: the default **Seq Scan** yields Contact 213 / user **830** (Bob), while the forced **Index Scan using `ix_contact_reply_email`** yields Contact 212 / user **829** (Alice). Neither plan is "wrong" — with no `ORDER BY`, both satisfy the query, and the planner's choice alone decides the recipient.

### Flip-back — reversible in both directions

A symmetric write to Contact 213 relocates *its* tuple, restoring Contact 212 as physically first:

```bash
docker exec -i sl-test-db psql -U test -d test < /tmp/obs_build/e_flipback.sql
# e_flipback.sql:
#   UPDATE contact SET updated_at = now() WHERE id = 213;
#   SELECT ctid, id, user_id FROM contact WHERE reply_email='re+shared@sl.local' ORDER BY ctid;
#   SELECT id, user_id FROM contact WHERE reply_email='re+shared@sl.local' LIMIT 1;
```
```
=== FLIP-BACK: UPDATE contact 213 -> relocates its tuple; 212 becomes physically first again ===
UPDATE 1
=== ctid after flip-back ===
  ctid  | id  | user_id 
--------+-----+---------
 (1,17) | 212 |     829
 (1,18) | 213 |     830
(2 rows)

=== what the unordered "... LIMIT 1" returns now ===
 id  | user_id 
-----+---------
 212 |     829
(1 row)
```
Contact 213 moved from `(1,16)` to `(1,18)`, so Contact 212 (user 829, Alice) is once again physically first at `(1,17)`; the unordered `... LIMIT 1` returns **212 / user 829** again.

### Run 3 — identical input, after flip-back

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/repro_loop.py 30 RUN3_after_flipback'
```
```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ctynzdudajgjaydwzzxv
Upload files to local dir
>>> init logging <<<
2026-07-08 06:04:08,282 - SL - DEBUG - 14742 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
======== DISTRIBUTION [RUN3_after_flipback] N=30 ========
  run  0 -> contact_id=212 user_id=829 [Alice(A)]
  run  1 -> contact_id=212 user_id=829 [Alice(A)]
  run  2 -> contact_id=212 user_id=829 [Alice(A)]
  run  3 -> contact_id=212 user_id=829 [Alice(A)]
  run  4 -> contact_id=212 user_id=829 [Alice(A)]
  run  5 -> contact_id=212 user_id=829 [Alice(A)]
  run  6 -> contact_id=212 user_id=829 [Alice(A)]
  run  7 -> contact_id=212 user_id=829 [Alice(A)]
  run  8 -> contact_id=212 user_id=829 [Alice(A)]
  run  9 -> contact_id=212 user_id=829 [Alice(A)]
  run 10 -> contact_id=212 user_id=829 [Alice(A)]
  run 11 -> contact_id=212 user_id=829 [Alice(A)]
  run 12 -> contact_id=212 user_id=829 [Alice(A)]
  run 13 -> contact_id=212 user_id=829 [Alice(A)]
  run 14 -> contact_id=212 user_id=829 [Alice(A)]
  run 15 -> contact_id=212 user_id=829 [Alice(A)]
  run 16 -> contact_id=212 user_id=829 [Alice(A)]
  run 17 -> contact_id=212 user_id=829 [Alice(A)]
  run 18 -> contact_id=212 user_id=829 [Alice(A)]
  run 19 -> contact_id=212 user_id=829 [Alice(A)]
  run 20 -> contact_id=212 user_id=829 [Alice(A)]
  run 21 -> contact_id=212 user_id=829 [Alice(A)]
  run 22 -> contact_id=212 user_id=829 [Alice(A)]
  run 23 -> contact_id=212 user_id=829 [Alice(A)]
  run 24 -> contact_id=212 user_id=829 [Alice(A)]
  run 25 -> contact_id=212 user_id=829 [Alice(A)]
  run 26 -> contact_id=212 user_id=829 [Alice(A)]
  run 27 -> contact_id=212 user_id=829 [Alice(A)]
  run 28 -> contact_id=212 user_id=829 [Alice(A)]
  run 29 -> contact_id=212 user_id=829 [Alice(A)]
  --- summary ---
  contact_id distribution: {212: 30}
  user_id    distribution: {829: 30}
  => contact 212 (user 829, Alice) = 30/30 ; contact 213 (user 830, Bob) = 0/30
```
→ flips back to **user 829** (Alice): 30/30. The behavior is **fully reversible** — the resolved user tracks physical order, which tracks benign writes.

### Honest characterization of the non-determinism

The resolution is **order-unspecified, not application-random**. Within a fixed *(heap-state, query-plan)* pair it is **100% stable per call** (every replay above was 30/30 to a single user). It **flips** between user 829 (Alice) and user 830 (Bob) across:
- routine DB writes that change physical order (`UPDATE` → MVCC relocation, insertion order, `VACUUM`), and
- query-plan choice (sequential scan vs. index scan).

This matches the user's report precisely: "**some** replies appear to be routed to the wrong user, even though the logs correctly show the alias being recognized." Recognition (`is_reverse_alias`, `app/email_utils.py:1158`) succeeds on the existence of either duplicate; the *identity* at `email_handler.py:986` may resolve to either row depending on physical order/plan — so across the fleet's varied DB states and query plans, a fraction of replies select the wrong contact and are relayed to the wrong user. The finding was reproduced and confirmed **stable across ≥2 runs** at each state (RUN1 = 30/30 Alice; RUN2 and RUN2b = 30/30 Bob; RUN3 = 30/30 Alice after flip-back), and is **fully reversible** in both directions.


---

## (f) Edge-branch observations (every condition, not just the happy path)

Every reply-phase decision branch was exercised at runtime through the **real** entry point by `/tmp/branch_test.py`, which builds a distinct `Envelope` + `EmailMessage` per scenario and calls `email_handler.handle(env, msg)` (which dispatches to `handle_reply(...)` at `email_handler.py:2199`), collecting the returned status string for each. All ten conditions were produced by a **single** run. The eight canonical branches (E200 happy, E501, E502×2, E503, E504, E214, E200-fallback) use the default config unchanged; the two non-canonical branches (E201, E506) monkeypatch a flag **only inside the ephemeral process** and restore it immediately after (no repository or config file is touched):

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/branch_test.py'
```

Complete, unedited output — the interleaved decision-log lines emitted by `handle_reply()` for each scenario, followed by the summary matrix printed by the script:

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/jejrcgaonlzlesrfkxdf
Upload files to local dir
>>> init logging <<<
2026-07-08 05:43:12,101 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,247 - SL - INFO - 14464 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:43:12,273 - SL - WARNING - 14464 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 05:43:12,282 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,287 - SL - WARNING - 14464 - "/app/email_handler.py:980" - handle_reply() -  - Reply email reply+e501obs@notmanaged-obs.test has wrong domain
2026-07-08 05:43:12,288 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,293 - SL - WARNING - 14464 - "/app/email_handler.py:988" - handle_reply() -  - No contact with reply+nonexistent-obs@sl.local as reverse alias
2026-07-08 05:43:12,293 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,300 - SL - WARNING - 14464 - "/app/email_handler.py:991" - handle_reply() -  - User <User 831 User Inactive user_inactive_obs@mailbox.test> has been soft deleted
2026-07-08 05:43:12,300 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,315 - SL - ERROR - 14464 - "/app/email_handler.py:1001" - handle_reply() -  - <Alias 1353 unmanaged_obs@baddomain-obs.test> domain isn't known
NoneType: None
2026-07-08 05:43:12,316 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,324 - SL - INFO - 14464 - "/app/app/models.py:888" - can_send_or_receive() -  - User <User 832 User Disabled user_disabled_obs@mailbox.test> is disabled. Cannot receive or send emails
2026-07-08 05:43:12,324 - SL - INFO - 14464 - "/app/email_handler.py:1008" - handle_reply() -  - User <User 832 User Disabled user_disabled_obs@mailbox.test> cannot send emails
2026-07-08 05:43:12,324 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,332 - SL - INFO - 14464 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:43:12,334 - SL - WARNING - 14464 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: stranger@not-authorized.test. msg from header: stranger@not-authorized.test, reverse-alias re+spoofon@sl.local, <Alias 1349 bonier_resits077@sl.local> <User 833 User SpoofOn user_spoofon_obs@mailbox.test> <Contact 216 spoofon_real@external.test 1349>
2026-07-08 05:43:12,358 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,367 - SL - INFO - 14464 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:43:12,367 - SL - WARNING - 14464 - "/app/email_handler.py:1023" - handle_reply() -  - ignore unknown sender to reverse-alias stranger@not-authorized.test: <Alias 1351 cotton_parade984@sl.local> -> <Contact 217 spoofoff_real@external.test 1351>
2026-07-08 05:43:12,383 - SL - WARNING - 14464 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 05:43:12,391 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,399 - SL - INFO - 14464 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:43:12,400 - SL - INFO - 14464 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:43:12,408 - SL - INFO - 14464 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:43:12,411 - SL - WARNING - 14464 - "/app/app/email_utils.py:804" - get_spam_from_header() -  - Spam score 10.0 exceeds 5
2026-07-08 05:43:12,412 - SL - WARNING - 14464 - "/app/email_handler.py:1081" - handle_reply() -  - Email detected as spam. Reply phase. <Alias 1351 cotton_parade984@sl.local> -> <Contact 217 spoofoff_real@external.test 1351>. Spam Score: None, Spam Report: None

================ REPLY-PHASE BRANCH MATRIX ================
scenario                               | rcpt_to (reply_email)        | mail_from                      | expect | actual status
--------------------------------------------------------------------------------------------------------------------------------------------
E200 happy (authorized mailbox)        | re+spoofoff@sl.local         | user_spoofoff_obs@mailbox.test | E200   | '250 Message accepted for delivery'
E501 wrong reply domain                | reply+e501obs@notmanaged-obs.test | user_e501_obs@mailbox.test     | E501   | '550 SL E501'
E502 no contact                        | reply+nonexistent-obs@sl.local | stranger@x.test                | E502   | '550 SL E502 Email not exist'
E502 inactive user (soft-deleted)      | re+inactive@sl.local         | user_inactive_obs@mailbox.test | E502   | '550 SL E502 Email not exist'
E503 alias domain unmanaged            | re+baddomain@sl.local        | user_baddomain_obs@mailbox.test | E503   | '550 SL E503'
E504 user cannot send/receive          | re+disabled@sl.local         | user_disabled_obs@mailbox.test | E504   | '550 SL E504 Account disabled'
E214 unauthorized (spoofing check ON)  | re+spoofon@sl.local          | stranger@not-authorized.test   | E214   | '250 SL E214 Unauthorized for using reverse alias'
E200 fallback (spoofing check OFF)     | re+spoofoff@sl.local         | stranger@not-authorized.test   | E200   | '250 Message accepted for delivery'
E201 SPF fail [NON-CANONICAL]          | re+spf@sl.local              | user_spf_obs@mailbox.test      | E201   | '250 SL E201'
E506 spam [NON-CANONICAL]              | re+spoofoff@sl.local         | user_spoofoff_obs@mailbox.test | E506   | '550 SL E506 Email detected as spam'
```

Every branch produced **exactly** its expected status. Default config flags for the canonical branches: `ENFORCE_SPF=False`, `ENABLE_SPAM_ASSASSIN=False`, `SPAMASSASSIN_HOST=None`, `DMARC_CHECK_ENABLED=True`, `MAX_REPLY_PHASE_SPAM_SCORE=5`.

The reply-phase status strings are defined in `app/email/status.py` (verbatim relevant lines):

```python
# app/email/status.py
E200 = "250 Message accepted for delivery"                     # :2
E201 = "250 SL E201"                                           # :3
E214 = "250 SL E214 Unauthorized for using reverse alias"      # :22
E501 = "550 SL E501"                                           # :38
E502 = "550 SL E502 Email not exist"                           # :39
E503 = "550 SL E503"                                           # :40
E504 = "550 SL E504 Account disabled"                          # :41
E506 = "550 SL E506 Email detected as spam"                    # :43
```

### Decision-code coverage (observed)

| # | Condition | Observed status | Guard `file:line` |
|---|-----------|-----------------|-------------------|
| 1 | Happy path (authorized `MAIL FROM`, single contact) | `250 Message accepted for delivery` (**E200**) | `app/email/status.py:2` |
| 2 | Wrong reply domain (`rcpt_to` not ending `EMAIL_DOMAIN`, no `SLDomain`) | `550 SL E501` (**E501**) | `email_handler.py:977-981` |
| 3 | No contact for `reply_email` | `550 SL E502 Email not exist` (**E502**) | `email_handler.py:987-989` |
| 4 | Inactive user (`is_active()` false — `delete_on` in the **future**) | `550 SL E502 Email not exist` (**E502**) | `email_handler.py:990-992` |
| 5 | User cannot send/receive (`user.disabled=True`) | `550 SL E504 Account disabled` (**E504**) | `email_handler.py:1007-1009` |
| 6 | Unknown mailbox, spoofing-check **on** | `250 SL E214 Unauthorized for using reverse alias` (**E214**) | `email_handler.py:1020-1034` (def `:1390`) |
| 7 | Unknown mailbox, spoofing-check **off** → fallback `mailbox = alias.mailbox` | `250 Message accepted for delivery` (**E200**) | `email_handler.py:1021-1029` |
| 8 | Invalid alias domain (alias domain not managed) | `550 SL E503` (**E503**) | `email_handler.py:999-1002` |
| 9 | Reply spam (**non-canonical**, see below) | `550 SL E506 Email detected as spam` (**E506**) | `email_handler.py:1094` |
| 10 | SPF fail (**non-canonical**, see below) | `250 SL E201` (**E201**) | `email_handler.py:1036-1040` |

**Both sides of the unknown-mailbox branch (verbatim):**

```python
# email_handler.py:1019-1034
    mailbox = get_mailbox_from_mail_from(mail_from, alias)
    if not mailbox:
        if alias.disable_email_spoofing_check:
            # ignore this error, use default alias mailbox
            LOG.w(
                "ignore unknown sender to reverse-alias %s: %s -> %s",
                mail_from,
                alias,
                contact,
            )
            mailbox = alias.mailbox
        else:
            # only mailbox can send email to the reply-email
            handle_unknown_mailbox(envelope, msg, reply_email, user, alias, contact)
            # return 2** to avoid Postfix sending out bounces and avoid backscatter issue
            return False, status.E214
```
- **Condition 6** used `alias.disable_email_spoofing_check=False` with `MAIL FROM=stranger@not-authorized.test` → `handle_unknown_mailbox(...)` then `return False, status.E214`.
- **Condition 7** used `alias.disable_email_spoofing_check=True` with the same unauthorized `MAIL FROM` → fell through to `mailbox = alias.mailbox` (`:1029`) and returned **E200**.

**Subtlety found at condition 4 (reported honestly):** `User.is_active()` (`app/models.py:766-769`) returns `delete_on < now`, so a user is **inactive only when `delete_on` is in the future**. An initial attempt with `delete_on` in the *past* left `is_active()` true and instead tripped **E504** via `can_send_or_receive()` (which rejects any non-null `delete_on`). Correcting the fixture to `delete_on = now + 30 days` produced the intended **E502** at `email_handler.py:990-992`. (The E502 checks are ordered: `is_active()` at `:990` runs before `can_send_or_receive()` at `:1007`.)

**Condition 9 (E506, reply spam) is labeled NON-CANONICAL.** Reply-spam detection requires `ENABLE_SPAM_ASSASSIN=True`, which is **off** in the default config, so this branch is inert canonically. To exercise it, `branch_test.py` set `email_handler.ENABLE_SPAM_ASSASSIN = True` for this one call (restored immediately after) and injected an `X-Spam-Status: Yes, score=10.0` header. The message reached `get_spam_info(msg, max_score=MAX_REPLY_PHASE_SPAM_SCORE)` at `email_handler.py:1076`, whose header parser `get_spam_from_header()` logged `Spam score 10.0 exceeds 5` at `app/email_utils.py:804`; `handle_reply()` then logged `Email detected as spam. Reply phase. <Alias 1351 cotton_parade984@sl.local> -> <Contact 217 spoofoff_real@external.test 1351>` at `email_handler.py:1081` and returned `status.E506` at `email_handler.py:1094` — observed as `'550 SL E506 Email detected as spam'` in the matrix above. Because it required a non-default flag, this value is explicitly **non-canonical**.

**Condition 10 (E201, SPF fail) is labeled NON-CANONICAL.** The guard is:

```python
# email_handler.py:1036-1040
    if ENFORCE_SPF and mailbox.force_spf and not alias.disable_email_spoofing_check:
        if not spf_pass(envelope, mailbox, user, alias, contact.website_email, msg):
            # cannot use 4** here as sender will retry.
            # cannot use 5** because that generates bounce report
            return True, status.E201
```
`ENFORCE_SPF=False` by default, so the branch is inert in canonical config. To exercise it **at runtime** (rather than merely infer it from the code), `branch_test.py` set `email_handler.ENFORCE_SPF = True` and `email_handler.spf_pass = lambda *a, **k: False` for this one call, then restored both. With the seeded `spf` mailbox (`mailbox.force_spf=True`, `alias.disable_email_spoofing_check=False`) the guard at `email_handler.py:1036` was satisfied, and the forced-failing `spf_pass` drove `return True, status.E201` at `email_handler.py:1040` — observed as `'250 SL E201'` in the matrix above. Because it required non-default flags plus a stubbed `spf_pass`, this value is explicitly **non-canonical** (the real path additionally needs a live DNS SPF failure).

### Normalization collision — how the collision surface is *widened*

`normalize_reply_email()` replaces every character outside its allow-list with `_` (verbatim):

```python
# app/email_validation.py:9
_ALLOWED_CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@"

# app/email_validation.py:25-38
def normalize_reply_email(reply_email: str) -> str:
    """Handle the case where reply email contains *strange* char that was wrongly generated in the past"""
    if not reply_email.isascii():
        reply_email = convert_to_id(reply_email)

    ret = []
    # drop all control characters like shift, separator, etc
    for c in reply_email:
        if c not in _ALLOWED_CHARS:
            ret.append("_")
        else:
            ret.append(c)

    return "".join(ret)
```

The repository's own test asserts the collapse (verbatim):

```python
# tests/test_email_utils.py:594-596
def test_normalize_reply_email(flask_client):
    assert normalize_reply_email("re+abcd@sl.local") == "re+abcd@sl.local"
    assert normalize_reply_email('re+"ab cd"@sl.local') == "re+_ab_cd_@sl.local"
```

Observed live via `/tmp/normalize.py`, which calls the real `normalize_reply_email()` (and `convert_to_id()` for the non-ascii case). Exact command and complete, unedited output:

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/normalize.py'
```
```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/chwmtnpgasyapktkkdhc
Upload files to local dir
>>> init logging <<<
2026-07-08 05:44:07,241 - SL - DEBUG - 14508 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
_ALLOWED_CHARS = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@'

== parity with tests/test_email_utils.py:594-596 ==
normalize_reply_email('re+abcd@sl.local')      = 're+abcd@sl.local'
normalize_reply_email('re+"ab cd"@sl.local')   = 're+_ab_cd_@sl.local'

== collision: distinct inbound addresses -> ONE normalized key ==
  'reply+a b@sl.local'     -> 'reply+a_b@sl.local'
  'reply+a"b@sl.local'     -> 'reply+a_b@sl.local'
  'reply+a,b@sl.local'     -> 'reply+a_b@sl.local'
  'reply+a;b@sl.local'     -> 'reply+a_b@sl.local'
  'reply+a(b@sl.local'     -> 'reply+a_b@sl.local'

  COLLISION: 5 distinct inputs all normalize to 'reply+a_b@sl.local':
       'reply+a b@sl.local'
       'reply+a"b@sl.local'
       'reply+a,b@sl.local'
       'reply+a;b@sl.local'
       'reply+a(b@sl.local'

== non-ascii path uses convert_to_id [app/utils.py:50] ==
normalize_reply_email('reply+caf\u00e9@sl.local') = 'reply+cafe@sl.local'
```

The first two lines reproduce the repository's own assertion at `tests/test_email_utils.py:594-596` exactly. The collision block then shows **five** distinct inbound addresses (differing by space, `"`, `,`, `;`, `(`) all folding to the single key `reply+a_b@sl.local`, and the non-ascii address `reply+café@sl.local` routing through `convert_to_id()` (`app/utils.py:50`) to `reply+cafe@sl.local`. Because normalization happens **before** the lookup (`email_handler.py:984` precedes `:986`), distinct reverse-aliases can be folded onto **one** key, which further **widens the collision surface** feeding the ambiguous `Contact.get_by(reply_email=...)` at `email_handler.py:986` — an additional, independent route to duplicate-key resolution.


---

## (g) The most-likely origin of the incorrect routing — cause → effect

**Named origin:** the single-sourced contact resolution at

```python
# email_handler.py:986
    contact = Contact.get_by(reply_email=reply_email)
```

executed over a **non-unique `reply_email`** column via a query with **no `ORDER BY`**.

This is the prime suspect because three independent facts combine:

1. **The query is unordered and limited to one row.** `ModelMixin.get_by` (`app/models.py:83-84`) is `filter_by(...).first()` → `... LIMIT 1` with no `ORDER BY` (SQL captured in section (e)). When ≥2 rows match, the returned row is unspecified.

2. **The database permits duplicate `reply_email` values.** The column has only a non-unique index (`app/models.py:1899` — `index=True`, **no `unique=True`**), created that way by the migration (verbatim):

   ```python
   # migrations/versions/2021_071310_78403c7b8089_.py:22
       op.create_index(op.f('ix_contact_reply_email'), 'contact', ['reply_email'], unique=False)
   ```

   The only `UniqueConstraint` on `Contact` covers a **different** pair and does not touch `reply_email`:

   ```python
   # app/models.py:1874-1876
       __table_args__ = (
           sa.UniqueConstraint("alias_id", "website_email", name="uq_contact"),
       )
   ```

   The application-level guard is **advisory only** — `available_sl_email()` itself just does another `Contact.get_by(reply_email=...)` with no DB constraint backing it, so a race or historical data can leave duplicates (verbatim):

   ```python
   # app/models.py:1425-1432
   def available_sl_email(email: str) -> bool:
       if (
           Alias.get_by(email=email)
           or Contact.get_by(reply_email=email)
           or DeletedAlias.get_by(email=email)
       ):
           return False
       return True
   ```

3. **Every downstream identity derives from that one row.** As shown in sections (c)/(d): `alias = contact.alias` (`:994`), `user = alias.user` (`:1004`), the authorizing mailbox via `get_mailbox_from_mail_from` (`:1019`/`:1364`), the logged `user_id = contact.user_id` (`:1046`), and the recipient `contact.website_email` (`:1121`).

**Cause → effect.** Because (2) allows two `Contact` rows to share one `reply_email`, and (1) makes the pick among them unspecified, the row chosen at `email_handler.py:986` can vary run-to-run without any input change (reproduced live in section (e): user 829 ⇄ user 830). Because (3) makes that one row the sole source of alias, user, mailbox, `user_id`, and recipient, a wrong-row pick mis-routes the **entire** reply to the wrong user. Meanwhile the alias is still "recognized," because dispatch's `is_reverse_alias()` (`app/email_utils.py:1158`) returns `True` on the mere **existence** of a matching row — decoupled from *which* row `:986` later selects. This is an exact match to the reported symptom: *"some replies appear to be routed to the wrong user, even though the logs correctly show the alias being recognized."*

**Security classification — this is a CONFIDENTIALITY defect, not just a mis-delivery.** A reverse-alias is intended to identify exactly one `(alias, contact)` pair so that a reply is relayed only between the owning user's mailbox and that user's own correspondent. When `:986` selects the duplicate belonging to a *different* user, three privacy-violating consequences follow **together**: (i) the reply body is delivered to the **other user's** contact address (`contact.website_email`, `email_handler.py:1121`), leaking one user's private correspondence to an unrelated third party; (ii) the `EmailLog` audit row is committed under the **other user's** `user_id` (`email_handler.py:1046`), misattributing the send; and (iii) the outgoing `From`/`To` are rewritten using the **other user's** alias and contact (`email_handler.py:1120-1122`), exposing that user's alias identity. The defect thus breaks the tenant-isolation guarantee that per-contact reverse-aliases exist to provide — the impact is disclosure of private data across accounts, which is why it should be treated as a confidentiality vulnerability rather than a cosmetic routing glitch.

**Contributing factor (widens the surface):** `normalize_reply_email()` (`app/email_validation.py:25`, called at `email_handler.py:984`) can fold **distinct** inbound reverse-aliases onto a **single** lookup key (section (f)), increasing the chance that `:986` matches more than one row.

### End-to-end data flow and the candidate mis-routing points

```mermaid
flowchart TD
    A["Inbound SMTP reply<br/>RCPT TO = reverse-alias reply_email"] --> B["MailHandler.handle_DATA<br/>email_handler.py:2289"]
    B --> C["_handle<br/>email_handler.py:2335"]
    C --> D["handle - central router<br/>email_handler.py:1945"]
    D --> E{"is_reverse_alias rcpt_to?<br/>email_handler.py:2195<br/>(Contact.get_by exists? email_utils.py:1158)"}
    E -- "No" --> F["handle_forward"]
    E -- "Yes = alias recognized" --> G["handle_reply<br/>email_handler.py:966"]
    G --> H["normalize_reply_email<br/>email_handler.py:984"]
    H --> I["contact = Contact.get_by reply_email .first<br/>email_handler.py:986"]
    I --> J["alias = contact.alias<br/>email_handler.py:994"]
    J --> K["user = alias.user<br/>email_handler.py:1004"]
    K --> L["mailbox = get_mailbox_from_mail_from<br/>email_handler.py:1019 / def 1364"]
    L --> M["EmailLog user_id = contact.user_id<br/>email_handler.py:1046"]
    M --> N["Deliver to contact.website_email<br/>email_handler.py:1121"]

    H -. "collapse to underscore widens collisions" .-> I
    I -. "PRIME: non-unique reply_email + first, no ORDER BY" .-> M
```

> **Out of scope (not implemented, per the read-only mandate):** this report **diagnoses** the origin; it does **not** remediate it. No `UniqueConstraint` was added to `Contact.reply_email`, no `ORDER BY`/tie-break was added to `Contact.get_by`/`ModelMixin.get_by`, `available_sl_email()` was not made atomic, and `generate_reply_email()`/`normalize_reply_email()` were not altered.


---

## (h) Reproduction fixtures and cleanup confirmation

### The fixture graph that recreated the precondition (seeded for observation, then removed)

Two independent identity graphs were seeded from an ephemeral `/tmp/seed.py` (developed outside the repo, `docker cp`'d into the container, run with `PYTHONPATH=/app`), following the proven pattern of `tests/test_email_handler.py::test_replace_contacts_and_user_in_reply_phase` (`:274`) and the helpers in `tests/utils.py` — but with the **same** `reply_email` across **distinct** users/aliases:

```bash
docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python /tmp/seed.py'
```

Seeded primary keys (recorded so they could be deleted afterward):

| Graph | user_id | alias_id (email) | mailbox_id | contact_id | website_email | reply_email |
|-------|---------|------------------|------------|------------|---------------|-------------|
| A | 829 | 1341 (`booked_reeves953@sl.local`) | 981 | 212 | `alice_real@external-A.test` | `re+shared@sl.local` |
| B | 830 | 1343 (`mutton_allude962@sl.local`) | 982 | 213 | `bob_real@external-B.test` | `re+shared@sl.local` |

Both aliases had `disable_email_spoofing_check=True`; both users had `replace_reverse_alias=True`; Contact 212 was inserted first (it holds the lower primary key), so at baseline it is the physically-earliest heap row and thus the `LIMIT 1` winner shown in section (e).

**Precondition proof — the schema permits two rows to share one `reply_email`:**

```bash
docker exec sl-test-db psql -U test -d test -c "SELECT id, alias_id, user_id, website_email, reply_email FROM contact WHERE reply_email='re+shared@sl.local' ORDER BY id;"
```
```
 id  | alias_id | user_id |       website_email        |    reply_email     
-----+----------+---------+----------------------------+--------------------
 212 |     1341 |     829 | alice_real@external-A.test | re+shared@sl.local
 213 |     1343 |     830 | bob_real@external-B.test   | re+shared@sl.local
(2 rows)
```

Two rows, **different** `alias_id` and `user_id`, **same** `reply_email` — the exact ambiguity that `Contact.get_by(reply_email=...).first()` at `email_handler.py:986` resolves non-deterministically. Additional branch fixtures for section (f) (users 831–837 — the inactive-user, disabled-alias, spoofing-on, spoofing-off, bad-reply-domain, SPF-fail, and E501 scenarios, with reply-emails `re+inactive@sl.local`, `re+disabled@sl.local`, `re+spoofon@sl.local`, `re+spoofoff@sl.local`, `re+baddomain@sl.local`, `re+spf@sl.local`, and `reply+e501obs@notmanaged-obs.test`) were seeded from `/tmp/seed.py` and `/tmp/seed_e501.py` and are likewise removed by the cleanup below.

### Cleanup — all seeded rows removed

The nine seeded users (`829`–`837`) owned 18 aliases, 9 contacts, 9 mailboxes, and 280 email-log rows (the email-log rows accumulated across the repeated drives and the identical-input replay loops in sections (b)–(f)). Foreign keys from child tables to `users` are `ON DELETE CASCADE`, so deleting the seeded users removes their aliases, contacts, mailboxes, and email-log rows in one step. The only non-cascade reference (`users.default_mailbox_id → mailbox`, `NO ACTION`) was nulled first:

```bash
docker exec -i sl-test-db psql -U test -d test -v ON_ERROR_STOP=1 <<'SQL'
BEGIN;
UPDATE users SET default_mailbox_id = NULL WHERE id BETWEEN 829 AND 837;
DELETE FROM users WHERE id BETWEEN 829 AND 837;
COMMIT;
SQL
```
```
BEGIN
UPDATE 9
DELETE 9
COMMIT
```

**Verification the mis-routing precondition rows are gone (0 rows):**

```bash
docker exec sl-test-db psql -U test -d test -c "SELECT id, alias_id, user_id, website_email, reply_email FROM contact WHERE reply_email='re+shared@sl.local';"
```
```
 id | alias_id | user_id | website_email | reply_email
----+----------+---------+---------------+-------------
(0 rows)
```

Post-delete counts for every seeded object (all zero), plus no orphan email-log rows:

```bash
docker exec sl-test-db psql -U test -d test -c "
SELECT 'users' AS tbl, count(*) FROM users WHERE id BETWEEN 829 AND 837
UNION ALL SELECT 'alias', count(*) FROM alias WHERE user_id BETWEEN 829 AND 837
UNION ALL SELECT 'contact', count(*) FROM contact WHERE user_id BETWEEN 829 AND 837
UNION ALL SELECT 'mailbox', count(*) FROM mailbox WHERE user_id BETWEEN 829 AND 837
UNION ALL SELECT 'email_log', count(*) FROM email_log WHERE user_id BETWEEN 829 AND 837;"
```
```
    tbl    | count
-----------+-------
 users     |     0
 alias     |     0
 contact   |     0
 mailbox   |     0
 email_log |     0
(5 rows)
```

All ephemeral observation scripts and their captured logs were removed. The eight Python drivers (`seed.py`, `seed_e501.py`, `drive_direct.py`, `drive_smtp.py`, `observe_chain.py`, `repro_loop.py`, `branch_test.py`, `normalize.py`) existed in both the container (`sl-app:/tmp/`) and the agent host; the section (e) SQL helpers (`e_show.sql` / `e_flip.sql` / `e_explain.sql` / `e_flipback.sql`) and the `e_driver.sh` wrapper lived only on the agent host and were piped into the container's `psql` over stdin. All of these scripts, together with the captured `.txt`/`.json` output logs, were deleted from both locations.

### Repository is byte-for-byte unchanged

```bash
git rev-parse --abbrev-ref HEAD                                  # current branch
git rev-parse HEAD                                               # HEAD before this document is committed
git status --porcelain                                           # every working-tree change
git status --porcelain -- . ':!blitzy/documentation/' | wc -l    # changes OUTSIDE the answer document
git submodule status | wc -l                                     # submodule count
```
```
blitzy-21ffb38f-7c02-40ed-ae41-d6a514f6966c
cc0866ab04c09cd301fce2dc0c54d6267be865b7
 M blitzy/documentation/app_2cd6ee777f8c.md
0
0
```

The **only** path that differs from `HEAD` is this answer document (`blitzy/documentation/app_2cd6ee777f8c.md`); the number of changes **outside** `blitzy/documentation/` is **0**, and the repository has no submodules. The identical invariant holds inside the container's `/app` bind-mount (same branch `blitzy-21ffb38f-7c02-40ed-ae41-d6a514f6966c`, `0` non-doc changes). **No tracked source, test, configuration, or migration file was modified, added, or deleted** — the entire investigation was performed through ephemeral `/tmp` scripts and temporary local-database rows, all of which have been removed (above). The `updated_at` bumps on contacts 212/213 during section (e) touched only those now-deleted DB rows, and the local dev database is not part of the git repository in any case. Committing this document therefore records exactly one changed path — the answer document itself — after which `git status` reports a clean working tree.


---

## Coverage pass — the user's 8 explicit asks, answered by name

1. **Use the development environment and simulate an inbound email reply.** Done — the canonical Python 3.10.18 stack (PostgreSQL 13, Redis 6, `EMAIL_DOMAIN=sl.local`, schema at head `32f25cbf12f6`) was used (section (a)), and an inbound reply to `re+shared@sl.local` was delivered through the real entry point via two canonical drives (section (b)).

2. **Trace the runtime flow end-to-end for that reply.** Done — `handle_DATA` (`email_handler.py:2289`) → `_handle` (`:2335`) → `handle` (`:1945`) → dispatch (`:2195`/`:2199`) → `handle_reply` (`:966`) → `normalize_reply_email` (`:984`) → `Contact.get_by` (`:986`) → `alias`/`user` (`:994`/`:1004`) → mailbox authorization (`:1019`) → `EmailLog.create` (`:1042-1050`) → delivery to `contact.website_email` (`:1121`), with captured logs (sections (b)–(d)).

3. **Observe which part of the system handles the incoming message.** Done — the aiosmtpd SMTP server in `email_handler.py` (`Controller(MailHandler())` at `:2383`), with `handle_DATA`/`_handle`/`handle` as the entry chain (section (b)).

4. **Observe how the alias is resolved to a user.** Done — single-sourced from one `Contact` row: `Contact.get_by(reply_email=...)` (`:986`) → `alias = contact.alias` (`:994`) → `user = alias.user` (`:1004`), with the mailbox-authorization step at `:1019` (section (c)); captured BEFORE/DURING values included.

5. **Observe what user ID the system ultimately forwards the reply to.** Done — `user_id = contact.user_id` written to `EmailLog` at `email_handler.py:1046`, delivered to `contact.website_email` at `:1121`; observed `user_id=829` → `alice_real@external-A.test` for the worked message (section (d)).

6. **Based on the live execution trace, explain the actual data flow.** Done — sections (b)–(e) present the captured traces and the single-sourced data flow, and section (e) reports the observed run-to-run distribution (user 829 ⇄ user 830) under identical input.

7. **Identify the most likely point where incorrect routing originates.** Done — `Contact.get_by(reply_email=...).first()` at `email_handler.py:986` over a non-unique `reply_email` (`app/models.py:1899`; migration `:22`) with no `ORDER BY` (`app/models.py:83-84`), with full cause → effect reasoning (section (g)).

8. **Treat temporary scripts/logs as permitted for observation but clean them up afterward.** Done — all `/tmp` scripts/logs removed and all seeded DB rows deleted; `git status` confirms the repository is byte-for-byte unchanged (section (h)).

---

## Appendix — consolidated `file:line` index

| Symbol / fact | `file:line` |
|---------------|-------------|
| `MailHandler.handle_DATA` | `email_handler.py:2289` |
| `MailHandler._handle` | `email_handler.py:2335` |
| `handle` (central router) | `email_handler.py:1945` |
| dispatch `is_reverse_alias(rcpt_to)` → `handle_reply(...)` | `email_handler.py:2195` → `:2199` |
| `main()` / `Controller(MailHandler())` | `email_handler.py:2381` / `:2383` |
| `handle_reply` (def) | `email_handler.py:966` |
| wrong reply domain → `E501` | `email_handler.py:977-981` |
| `normalize_reply_email(reply_email)` call | `email_handler.py:984` |
| **`contact = Contact.get_by(reply_email=...)`** (PRIME) | `email_handler.py:986` |
| no contact / inactive user → `E502` | `email_handler.py:987-989` / `:990-992` |
| `alias = contact.alias` | `email_handler.py:994` |
| invalid alias domain → `E503` | `email_handler.py:999-1002` |
| `user = alias.user` | `email_handler.py:1004` |
| user cannot send/receive → `E504` | `email_handler.py:1007-1009` |
| `mailbox = get_mailbox_from_mail_from(...)` (call / def) | `email_handler.py:1019` / `:1364` |
| disabled-spoofing fallback `mailbox = alias.mailbox` | `email_handler.py:1021-1029` |
| unknown mailbox → `handle_unknown_mailbox` / `E214` (def) | `email_handler.py:1032` / `:1034` (def `:1390`) |
| SPF enforcement → `E201` | `email_handler.py:1036-1040` |
| `EmailLog.create(..., user_id=contact.user_id, mailbox_id=mailbox.id, ...)` | `email_handler.py:1042-1050` (`user_id` at `:1046`) |
| reply spam → `E506` | `email_handler.py:1094` |
| deliver to `contact.website_email` | `email_handler.py:1121` |
| `ModelMixin.get_by` = `filter_by(...).first()` (no `ORDER BY`) | `app/models.py:83-84` |
| `available_sl_email()` (advisory) | `app/models.py:1425` |
| `uq_contact(alias_id, website_email)` | `app/models.py:1874-1876` |
| `Contact.reply_email` (index, not unique) | `app/models.py:1899` |
| `is_reverse_alias()` (recognizes on existence) | `app/email_utils.py:1156-1163` |
| `normalize_reply_email()` / `_ALLOWED_CHARS` | `app/email_validation.py:25` / `:9` |
| `convert_to_id()` | `app/utils.py:50` |
| `create_contact()` / `reply_email=` assignment | `app/contact_utils.py:42` / `:89`, `:97` |
| `ix_contact_reply_email` created `unique=False` | `migrations/versions/2021_071310_78403c7b8089_.py:22` |
| status codes `E200/E201/E214/E501/E502/E503/E504/E506` | `app/email/status.py:2,3,22,38,39,40,41,43` |
| normalization collapse test | `tests/test_email_utils.py:594-596` |
| mail-capture hook (`get_stored_emails` / decorator) | `app/mail_sender.py:108` / `:111` |
| reference reply fixture pattern (two contacts) | `tests/test_email_handler.py:274` |
| canonical DB connection string | `tests/test.env:17` |

