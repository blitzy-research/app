# SimpleLogin Email-Forward Investigation — Runtime Evidence Report

**Reported production issue:** *"inconsistent behavior when emails are forwarded through SimpleLogin aliases."*

This report answers four precise questions by **running an actual email-forward operation through the real SimpleLogin email handler and capturing runtime values** — not values inferred from reading the code alone. Every value below is grounded with the exact command that produced it, the complete unedited output, and a `file:line` reference into the source at commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`, and is explicitly labeled **[OBSERVED]** (captured from a live run) or **[INFERRED]** (explained from source, not directly observed).

The four questions:

- **Q1 — Log message text (success vs. non-existent alias):** the exact log text and level emitted when an email is (a) successfully forwarded and (b) fails because the target alias does not exist.
- **Q2 — SL Message-ID vs. original Message-ID:** the specific "SL Message-ID" generated during forwarding and how it differs from the sender's original `Message-ID`.
- **Q3 — Transformed `From` header and reply-email format:** the exact `From` header value of the forwarded email after transformation, including the reverse-alias (reply-email) address format.
- **Q4 — Database records created by a single forward:** every database record created during one forward operation, with actual record IDs and timestamps.

---

## 0. TL;DR (all claims labeled; conclusions bounded to what was observed)

| # | Answer (summary) | Label |
|---|------------------|-------|
| Q1 | **Success** ends with the `LOG.i` summary line whose tail reads `with return code '250 Message accepted for delivery'<<===` (`email_handler.py:2367`, status `E200` at `app/email/status.py:2`). **Non-existent alias** emits `LOG.d` `alias <x> not exist. Try to see if it can be created on the fly` (`email_handler.py:545`), then `LOG.d` `alias <x> cannot be created on-the-fly, return 550` (`email_handler.py:551`), then the `LOG.i` summary line whose tail reads `with return code '550 SL E515 Email not exist'<<===` (status `E515` at `app/email/status.py:51`). Complete lines in §4.1 / §4.2. | [OBSERVED] |
| Q2 | On a **forward**, the original `Message-ID` header is **preserved unchanged** — delivered `Message-ID` equals the sent one, byte-for-byte (0-byte diff across 3 identical runs). The distinct `sl_message_id` produced by `make_msgid(...)` (`email_handler.py:1311`) is a **reply-phase** artifact persisted in `message_id_matching` (`app/models.py:3365`); it was **not created by any forward** (table stayed empty; `email_log.sl_message_id` stayed empty). A third, separate identifier — the per-message log-tracing id — is a `uuid4` (`email_handler.py:2339`) that varies every message. | [OBSERVED] + [INFERRED] disambiguation |
| Q3 | Delivered `From` after transformation: `"hey at google.com" <hey_at_google_com_pfxrzq@sl.local>` — display name is the sender with `@`→` at ` (`SenderFormatEnum.AT`, `app/models.py:2028-2029`); the reverse-alias local part is `{sanitized_sender}_{random}` with a prefix-less random suffix of 5–10 chars (`generate_reply_email`, `app/email_utils.py:1138-1143`). | [OBSERVED] |
| Q4 | A **first** forward from a new sender creates exactly three rows: `Contact` (id 2), `UserAuditLog` (id 1, action `create_contact`), `EmailLog` (id 2). A **repeat** forward from the same sender reuses the `Contact` and creates only a new `EmailLog` (ids 3, then 4). `message_id_matching` is never written on forward. | [OBSERVED] |

**Bounded conclusion [OBSERVED for the forward path; HYPOTHESIS beyond it]:** across three byte-identical replays of the same input, the forward path produced **stable** log templates, SMTP status codes, the preserved `Message-ID`, and the `From`/reverse-alias **format**; it produced **variable** `EmailLog.id`, `created_at` timestamps, the log-tracing `uuid4`, and the delivered-message VERP envelope-from. The most likely explanation of a *perceived* "inconsistency" is (i) these per-run-variable fields and (ii) the **phase-dependent** Message-ID handling (preserved on forward vs. replaced on reply). This is offered as a **hypothesis**: the reply path and bounce path were not exercised in this investigation, so this report does **not** assert the reported issue is "not a defect" or globally deterministic — only that the **forward** path behaved as recorded here.

---

## 1. Evidence conventions & scope discipline

- **Canonical entry point only.** Every message is injected over real SMTP into the `aiosmtpd` controller that `email_handler.py` starts on `127.0.0.1:20381` (`email_handler.py:2383`, `:2386`, `:2403`), using `swaks`. No value in this report comes from a direct Python call to `handle_forward()`/`forward_email_to_mailbox()`; there are **no non-canonical / bypassing values**.
- **Byte-identical replay.** To characterize run-to-run behavior the *same unchanged bytes* are replayed (a single fixed DATA payload with exactly one `Message-ID`); the report presents the observed distribution rather than a stabilized variant.
- **Complete unedited output.** Command outputs are shown in full inside fenced blocks; no ellipses, tails, or paraphrase stand in for real output.
- **Read-only against the product.** No file in the SimpleLogin source tree was modified. The only persistent artifact produced by this task is this document. All temporary scripts, captured `.eml` dumps, and the investigation container/services are torn down (Section 11).
- **Labels.** **[OBSERVED]** = captured from the live run whose command is shown; **[INFERRED]** = explained from the source at the cited `file:line`, not directly observed at runtime.
- **Reading the `psql` output.** Every database query uses `psql … -tAc "<SQL>"`: `-t` = tuples-only (no header/footer), `-A` = unaligned output, `-c` = run the single command and exit. In unaligned mode `psql` separates selected columns with the default field separator `|`, so a row such as `2|5|hey@google.com|hey_at_google_com_pfxrzq@sl.local|2026-07-13 17:54:36.152853` is exactly the selected columns in order, `|`-delimited. The selected columns in each query match the columns described alongside it.

---

## 2. Environment & reproducibility transcript

### 2.1 Host vs. container boundary

The canonical runtime is the user-specified Docker image (Python 3.10 + Postgres + Redis). Docker runs on the **host**; **all** investigation steps run **inside the container** via `docker exec`, with the working directory `/app` (the SimpleLogin checkout). The deliverable document is written on the **host** repository working tree. The image entrypoint is `/bin/bash`, so the container is started with `--entrypoint bash <image> -lc "sleep infinity"` (exact command below) and driven with `docker exec`.

**Container provisioning (host command):**

```bash
INV_NAME="sl_inv_06d10137_$(date +%s)"
IMG="ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0"
CID=$(docker run -d --name "$INV_NAME" \
        --label blitzy-investigation=app_2cd6ee777f8c \
        --entrypoint bash "$IMG" -lc "sleep infinity")
echo "$CID"    # full container id, recorded for safe teardown
```

Observed container identity **[OBSERVED]**:

```
name: sl_inv_06d10137_1783964622
id:   0e8a6464d53930be6167ac1e53a39087111e2531212e38dafa4cfb591c076967
label: blitzy-investigation=app_2cd6ee777f8c
```

### 2.2 Runtime & source commit (inside container)

Command:

```bash
docker exec "$CID" bash -lc 'cd /app && /app/venv/bin/python --version && echo "git HEAD: $(git rev-parse HEAD)"'
```

Output **[OBSERVED]**:

```
Python 3.10.18
git HEAD: 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
```

The runtime is Python 3.10.18 (matching `pyproject.toml:61` `python = "^3.10"`), and the source is exactly the reviewed commit.

### 2.3 Required one-time fix: `pyre2` replaces `google-re2`

The image venv ships `google-re2` (`google-re2==1.1.20250805`), whose `re2` module lacks `DOTALL`/`IGNORECASE` (`getattr(re2, "DOTALL", "MISSING")` → `MISSING` **[OBSERVED]**). On a **canonical** checkout — where `app/spamassassin_utils.py:8` is `import re2 as re` — importing the handler therefore fails at import time, because the module-level `re.compile(rb"...", re.DOTALL)` at `app/spamassassin_utils.py:13` dereferences that missing attribute. The failing import chain is `email_handler.py:92` (`from app.email.spam import get_spam_score`) → `app/email/spam.py:11` (`from app.spamassassin_utils import SpamAssassin`) → `app/spamassassin_utils.py:13`. Captured traceback **[OBSERVED]** (canonical source with `google-re2` installed, run before applying the fix below):

```
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/app/email_handler.py", line 92, in <module>
    from app.email.spam import get_spam_score
  File "/app/app/email/spam.py", line 11, in <module>
    from app.spamassassin_utils import SpamAssassin
  File "/app/app/spamassassin_utils.py", line 13, in <module>
    divider_pattern = re.compile(rb"^(.*?)\r?\n(.*?)\r?\n\r?\n", re.DOTALL)
AttributeError: module 're2' has no attribute 'DOTALL'
```

Installing the lock-consistent `pyre2` resolves it. This is a **setup** correction to the environment, not a change to any product file.

> **Image-baseline caveat [OBSERVED]:** the image's in-container `/app/app/spamassassin_utils.py` carries a baseline patch that changes line 8 to the stdlib `import re` (which *does* expose `DOTALL`), so the import failure is masked inside this particular image even before the `pyre2` fix. The canonical source — and the host deliverable checkout at this commit — use `import re2 as re`, so a canonical checkout fails exactly as shown above; the traceback above was captured after restoring line 8 to the canonical `import re2 as re` in the disposable container. `pyre2` is the correct `poetry.lock`-consistent remedy regardless (`pyproject.toml` requires `pyre2 = "^0.3.6"`).

Fix command:

```bash
docker exec "$CID" bash -lc '/app/venv/bin/pip uninstall -y google-re2 && \
  /app/venv/bin/pip install --only-binary :all: pyre2==0.3.10'
```

Verification command:

```bash
docker exec "$CID" bash -lc 'cd /app && /app/venv/bin/python -c "import re2; print(\"re2 module file:\", re2.__file__); print(\"DOTALL:\", getattr(re2,\"DOTALL\",\"MISSING\")); print(\"IGNORECASE:\", getattr(re2,\"IGNORECASE\",\"MISSING\"))"'
docker exec "$CID" bash -lc '/app/venv/bin/pip freeze | grep -iE "^(pyre2|google-re2)" || echo "(none)"'
```

Output **[OBSERVED]**:

```
re2 module file: /app/venv/lib/python3.10/site-packages/re2.cpython-310-x86_64-linux-gnu.so
DOTALL: re.DOTALL
IGNORECASE: re.IGNORECASE
pyre2==0.3.10
```

`re2` is now the compiled `pyre2` extension exposing `DOTALL`/`IGNORECASE`, and `google-re2` is absent. `import email_handler` then succeeds:

```bash
docker exec "$CID" bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/python -c "import email_handler; print(\"import email_handler OK\")"'
```

Output **[OBSERVED]** (trailing lines; full app-config banner precedes):

```
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/wsqmanwhqmmhyzwnfyli
Upload files to local dir
>>> init logging <<<
2026-07-13 18:04:13,394 - SL - DEBUG - 1263 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
import email_handler OK
```

### 2.4 Services: Postgres, Redis, SMTP sink

- **Postgres 15.13** on port `15432` (the cluster's `postgresql.conf` `port` set to `15432`; started with `pg_ctlcluster 15 main start`), role `myuser` (SUPERUSER, password `mypassword`), database `simplelogin` owned by `myuser`. The `pg_trgm` extension is **not** pre-created — it is created by the migration on a clean database (pre-creating it makes migration `2021_082012_424808e1fe49` roll back on `DUPLICATE_OBJECT`).
- **Redis 7.x** on `6379` (`redis-server --daemonize yes --port 6379`).
- **SMTP sink** on `127.0.0.1:1025` (Section 2.5).

Readiness commands and output **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "select '"'"'pg_ok'"'"', count(*) from pg_stat_activity;" | head -1'
docker exec "$CID" bash -lc 'redis-cli -p 6379 ping'
docker exec "$CID" bash -lc 'python3 - <<PY
import socket
s=socket.create_connection(("127.0.0.1",1025),timeout=3)
print("sink banner:", s.recv(100).decode().strip())
s.close()
PY'
```

```
pg_ok|7
PONG
sink banner: 220 0e8a6464d539 Python SMTP 1.4.2
```

### 2.5 SMTP sink: complete published source and justification (nonstandard-vs-MailHog)

The Agent Action Plan suggests MailHog / mailcatcher as the downstream sink. **MailHog, mailcatcher, and the Go toolchain are absent from the canonical image**, so the sink is built on **`aiosmtpd`**, which is **already a pinned project dependency** (`pyproject.toml`: `aiosmtpd ^1.2`; installed `1.4.2`). This is therefore a **nonstandard sink relative to MailHog** but uses a first-party, already-present dependency and performs **zero transformation** — it writes each accepted message's exact `envelope.content` bytes verbatim to `CAPTURE_DIR/msg_NNNN.eml`, so the delivered `From` and `Message-ID` can be read byte-for-byte. The complete source is published here (`sha256=b7dccb4ead4b9c8ed3b8c681946e358faffa424fc0935df375ba6c57d8979ac3`):

```python
#!/usr/bin/env python3
"""
blitzy_inv_sink.py - minimal downstream SMTP sink for the READ-ONLY
SimpleLogin email-forward investigation (temporary; deleted afterward).

WHY THIS SINK (justification for finding of nonstandard sink):
  MailHog / mailcatcher (and the Go toolchain) are ABSENT from the
  canonical image, so this sink is built on `aiosmtpd`, which is ALREADY
  a pinned project dependency (pyproject.toml: aiosmtpd ^1.2; installed
  1.4.2). It writes each accepted message's EXACT DATA payload
  (envelope.content) verbatim, with zero transformation, to
  CAPTURE_DIR/msg_NNNN.eml, so the delivered `From` and `Message-ID`
  can be read byte-for-byte. It is a pure sink: it stores and accepts.

LISTENS: 127.0.0.1:1025  (matches POSTFIX_SERVER=localhost / POSTFIX_PORT=1025)
"""
import os
import time
from aiosmtpd.controller import Controller

CAPTURE_DIR = os.environ.get("SINK_CAPTURE_DIR", "/tmp/blitzy_inv_sink_capture")
HOST = "127.0.0.1"
PORT = 1025


class RawStoringSink:
    """aiosmtpd message-storing handler: persists raw bytes, accepts with 250."""

    def __init__(self):
        self.count = 0

    async def handle_DATA(self, server, session, envelope):
        self.count += 1
        path = os.path.join(CAPTURE_DIR, "msg_%04d.eml" % self.count)
        with open(path, "wb") as fh:
            fh.write(envelope.content)  # exact raw delivered bytes, no rewrite
        print(
            "SINK stored #%d mail_from=%s rcpt_tos=%s bytes=%d -> %s"
            % (self.count, envelope.mail_from, envelope.rcpt_tos,
               len(envelope.content), path),
            flush=True,
        )
        return "250 Message accepted for delivery"


def main():
    os.makedirs(CAPTURE_DIR, exist_ok=True)
    controller = Controller(RawStoringSink(), hostname=HOST, port=PORT)
    controller.start()  # serves in a background thread
    print("SINK ready on %s:%d capture_dir=%s" % (HOST, PORT, CAPTURE_DIR),
          flush=True)
    while True:
        time.sleep(1)


if __name__ == "__main__":
    main()
```

Startup and the complete run-log (each stored message → file) **[OBSERVED]**:

```
SINK ready on 127.0.0.1:1025 capture_dir=/tmp/blitzy_inv_sink_capture
SINK stored #1 mail_from=sl.lmycyibsfqqdemzygi4dgnc5.e6al3q2pm6lsa@sl.local rcpt_tos=['john@wick.com'] bytes=533 -> /tmp/blitzy_inv_sink_capture/msg_0001.eml
SINK stored #2 mail_from=sl.lmycyibtfqqdemzygi4dgnk5.vyb6ku3x73gks@sl.local rcpt_tos=['john@wick.com'] bytes=533 -> /tmp/blitzy_inv_sink_capture/msg_0002.eml
SINK stored #3 mail_from=sl.lmycyibufqqdemzygi4dgnk5.lb2s6zhltoi2c@sl.local rcpt_tos=['john@wick.com'] bytes=533 -> /tmp/blitzy_inv_sink_capture/msg_0003.eml
SINK stored #4 mail_from=sl.lmycyibvfqqdemzygi4dgn25.e645nmdcc6a5i@sl.local rcpt_tos=['john@wick.com'] bytes=433 -> /tmp/blitzy_inv_sink_capture/msg_0004.eml
SINK stored #5 mail_from=sl.lmycyibwfqqdemzygi4dgn25.dc6zl2ihcp3w6@sl.local rcpt_tos=['john@wick.com'] bytes=426 -> /tmp/blitzy_inv_sink_capture/msg_0005.eml
SINK stored #6 mail_from=sl.lmycyibxfqqdemzygi4dgn25.gxxh36h5xaijs@sl.local rcpt_tos=['john@wick.com'] bytes=437 -> /tmp/blitzy_inv_sink_capture/msg_0006.eml
```

Messages `#1–#3` are the three byte-identical `hey@google.com → e1@sl.local` runs (Sections 4–7); `#4–#6` are the Q3 new-sender sub-experiment (Section 6.3). The non-existent-alias failure (Section 4.2) stored **nothing** — proof of no downstream egress.

### 2.6 Configuration: `.env` deltas and resolution

`.env` is derived from `example.env` with exactly four deltas so a real forward is emitted to the local sink. Command and output **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'cd /app && diff example.env .env'
```

```
19c19
< NOT_SEND_EMAIL=true
---
> # NOT_SEND_EMAIL=true  # commented out: presence-based; real forward required
69c69
< # POSTFIX_SERVER=my-postfix.com
---
> POSTFIX_SERVER=localhost
75c75
< DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
---
> DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
154c154
< # POSTFIX_PORT=1025
---
> POSTFIX_PORT=1025
```

`NOT_SEND_EMAIL` and `ENABLE_SPAM_ASSASSIN` are **presence-based** flags (`app/config.py:91`, `:450`): commenting `NOT_SEND_EMAIL` out makes it resolve to `False`, enabling a genuine send. Resolved configuration **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/python -c "
from app import config
print(\"NOT_SEND_EMAIL =\", config.NOT_SEND_EMAIL)
print(\"POSTFIX_SERVER =\", config.POSTFIX_SERVER)
print(\"POSTFIX_PORT   =\", config.POSTFIX_PORT)
print(\"EMAIL_DOMAIN   =\", config.EMAIL_DOMAIN)
print(\"DB_URI         =\", config.DB_URI)
"'
```

```
NOT_SEND_EMAIL = False
POSTFIX_SERVER = localhost
POSTFIX_PORT   = 1025
EMAIL_DOMAIN   = sl.local
DB_URI         = postgresql://myuser:mypassword@localhost:15432/simplelogin
```

### 2.7 Schema migration and fixture seed

Migration `CONFIG=/app/.env alembic upgrade head` exited `0` and produced 265 log lines: the complete 10-line application/alembic banner (shown below verbatim) followed by exactly **255 sequential** `Running upgrade` transition lines. Rather than paste 255 near-identical transition lines, the complete banner plus the first and last transitions are shown verbatim, and migration completion is then proven by a **hard post-migration state check** (head revision, table count, `pg_trgm`) — no substantive output is elided behind an ellipsis.

Complete banner and boundary transitions **[OBSERVED]**:

```
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/vvfibcigcjmnjtdguand
Upload files to local dir
>>> init logging <<<
2026-07-13 17:47:52,363 - SL - DEBUG - 565 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

(The first `Running upgrade  -> 5e549314e1e2` and the final three transitions are the boundaries of the 255-line sequence; the intervening 251 transitions are the remaining sequential `Running upgrade X -> Y` lines and carry no distinct evidentiary value.)

Post-migration state verification **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/alembic current 2>&1 | tail -1'
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select '"'"'public_tables'"'"', count(*) from information_schema.tables where table_schema='"'"'public'"'"'
union all select '"'"'pg_trgm_installed'"'"', count(*) from pg_extension where extname='"'"'pg_trgm'"'"';"'
```

```
32f25cbf12f6 (head)
public_tables|77
pg_trgm_installed|1
```

Head is `32f25cbf12f6`; there are 77 public tables; and `pg_trgm` is installed (created **by** the migration, not pre-created).

Seed `CONFIG=/app/.env FLASK_APP=wsgi:app flask dummy-data` — exit 0, complete 27-line output **[OBSERVED]**:

```
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/xklezmptvpxidtjovxmo
Upload files to local dir
>>> init logging <<<
2026-07-13 17:48:07,607 - SL - DEBUG - 608 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 17:48:08,379 - SL - WARNING - 608 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-13 17:48:08,379 - SL - DEBUG - 608 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-13 17:48:08,707 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:08,711 - SL - DEBUG - 608 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 17:48:08,730 - SL - DEBUG - 608 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_list596@sl.local
2026-07-13 17:48:08,741 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:08,793 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:08,801 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:08,819 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:08,830 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:08,845 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:08,853 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:08,863 - SL - DEBUG - 608 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-wwdfadtvig
2026-07-13 17:48:08,870 - SL - DEBUG - 608 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-mflwazwwty
2026-07-13 17:48:09,144 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:09,146 - SL - DEBUG - 608 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 17:48:09,168 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:09,181 - SL - INFO - 608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:48:09,191 - SL - INFO - 608 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

**Seeded fixtures used by the reproduction [OBSERVED]:** user `john@wick.com` (id 1, `include_sender_in_reverse_alias = t`); alias `e1@sl.local` = **alias id 5** (the success target). A pre-seeded baseline `Contact` (id 1) is `hey@google.com` on a *different, random* alias (alias_id 2), so the first `hey@google.com → e1@sl.local` forward creates a **new** contact (relevant to Q4).

### 2.8 Starting the canonical entry point

Command (inside container):

```bash
nohup env CONFIG=/app/.env /app/venv/bin/python email_handler.py >/tmp/handler.log 2>&1 &
```

Startup lines **[OBSERVED]**:

```
Listen for port 20381
Start mail controller 0.0.0.0 20381
```

These correspond to `LOG.i("Listen for port %s", ...)` (`email_handler.py:2403`) and `"Start mail controller %s %s"` (`email_handler.py:2386`); the argparse default port is `20381` (`email_handler.py:2399`). The handler is now reachable at `127.0.0.1:20381` — the real, canonical SMTP interface.

---

## 3. The fixed, byte-identical input (basis for Q1, Q2, and stability)

To separate stable from run-variable behavior, a **single fixed DATA payload** with **exactly one** `Message-ID` header is replayed unchanged. It was built once and hashed. Command and output **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'cat -A /tmp/payload.eml'      # -A shows CRLF as ^M$
docker exec "$CID" bash -lc 'sha256sum /tmp/payload.eml; echo "bytes: $(wc -c < /tmp/payload.eml)"'
```

```
Date: Mon, 13 Jul 2026 12:00:00 +0000^M$
To: e1@sl.local^M$
From: hey@google.com^M$
Subject: Blitzy fixed forward probe^M$
Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>^M$
Content-Type: text/plain; charset=UTF-8^M$
Content-Transfer-Encoding: 7bit^M$
^M$
Fixed byte-identical body for run-to-run replay. Do not change.^M$
c1dd2e00b7fc4009815f7b4a13b61151c367744ec480c5cfa2e599daf1f84471  /tmp/payload.eml
bytes: 314
```

The payload is `314` bytes, CRLF-terminated, with exactly one `Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>`. All three success runs inject this **same file** via:

```bash
swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 --data - < /tmp/payload.eml
```

**Byte-identity of what the handler actually received [OBSERVED].** The handler logs the parsed inbound envelope on one line (`==>> Handle ...`, `email_handler.py:1980`). Hashing that exact line for each of the three runs yields the identical digest, proving the handler processed identical input every time:

```bash
for r in run1 run2 run3; do
  line=$(grep -m1 '==>> Handle' ${r}_handler.txt | sed 's/^.*==>> Handle/==>> Handle/')
  printf "%s inbound-line-sha256: %s\n" "$r" "$(printf '%s' "$line" | sha256sum | cut -d' ' -f1)"
done
```

```
run1 inbound-line-sha256: ec07697b1bb5eda1819cc7d42ac66bed8757f892a0bc36fcb7a613d42e96dabc
run2 inbound-line-sha256: ec07697b1bb5eda1819cc7d42ac66bed8757f892a0bc36fcb7a613d42e96dabc
run3 inbound-line-sha256: ec07697b1bb5eda1819cc7d42ac66bed8757f892a0bc36fcb7a613d42e96dabc
```

Identical digest across all three runs ⇒ **byte-identical input**, satisfying the "replay the SAME unchanged input" discipline.

---

## 4. Q1 — Log message text (success vs. non-existent alias)

### 4.1 Success — complete handler stdout (run #1, first send)

Command (host) that injected run #1:

```bash
swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 --data - < /tmp/payload.eml
```

Complete SMTP transcript returned to the client **[OBSERVED]** — terminal status is `250 Message accepted for delivery`:

```
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 0e8a6464d539 Python SMTP 1.4.2
 -> EHLO 0e8a6464d539
<-  250-0e8a6464d539
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<hey@google.com>
<-  250 OK
 -> RCPT TO:<e1@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Mon, 13 Jul 2026 12:00:00 +0000
 -> To: e1@sl.local
 -> From: hey@google.com
 -> Subject: Blitzy fixed forward probe
 -> Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
 -> Content-Type: text/plain; charset=UTF-8
 -> Content-Transfer-Encoding: 7bit
 -> 
 -> Fixed byte-identical body for run-to-run replay. Do not change.
 -> 
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

Complete handler stdout for run #1 **[OBSERVED]** (unedited; log-tracing id `83d50cd9-5687-4f5c-b787-025bbabb0b12`):

```
2026-07-13 17:54:35,978 - SL - DEBUG - 671 - "/app/app/log.py:24" - set_message_id() -  - set message_id 83d50cd9-5687-4f5c-b787-025bbabb0b12
2026-07-13 17:54:35,978 - SL - DEBUG - 671 - "/app/email_handler.py:2342" - _handle() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - ====>=====>====>====>====>====>====>====>
2026-07-13 17:54:35,979 - SL - INFO - 671 - "/app/email_handler.py:2343" - _handle() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - New message, mail from hey@google.com, rctp tos ['e1@sl.local'] 
2026-07-13 17:54:35,980 - SL - DEBUG - 671 - "/app/email_handler.py:1963" - handle() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Cannot parse Postfix queue ID from None None
2026-07-13 17:54:36,116 - SL - DEBUG - 671 - "/app/email_handler.py:1980" - handle() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - ==>> Handle mail_from:hey@google.com, rcpt_tos:['e1@sl.local'], header_from:hey@google.com, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<blitzy-fixed-probe-2cd6ee777f8c@test.local>, client_ip:None, headers:[('Date', 'Mon, 13 Jul 2026 12:00:00 +0000'), ('To', 'e1@sl.local'), ('From', 'hey@google.com'), ('Subject', 'Blitzy fixed forward probe'), ('Message-ID', '<blitzy-fixed-probe-2cd6ee777f8c@test.local>'), ('Content-Type', 'text/plain; charset=UTF-8'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 17:54:36,121 - SL - DEBUG - 671 - "/app/email_handler.py:2202" - handle() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Forward phase hey@google.com(hey@google.com) -> e1@sl.local
2026-07-13 17:54:36,137 - SL - DEBUG - 671 - "/app/email_handler.py:580" - handle_forward() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Create or get contact for from_header:hey@google.com
2026-07-13 17:54:36,162 - SL - DEBUG - 671 - "/app/app/contact_utils.py:110" - create_contact() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Created contact <Contact 2 hey@google.com 5> for alias <Alias 5 e1@sl.local> with email hey@google.com invalid_email=False
2026-07-13 17:54:36,162 - SL - INFO - 671 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - DMARC check disabled
2026-07-13 17:54:36,170 - SL - DEBUG - 671 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Forward <Contact 2 hey@google.com 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-13 17:54:36,173 - SL - DEBUG - 671 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Create <EmailLog 2> for <Contact 2 hey@google.com 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-13 17:54:36,179 - SL - DEBUG - 671 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - From header, new:"hey at google.com" <hey_at_google_com_pfxrzq@sl.local>, old:hey@google.com
2026-07-13 17:54:36,179 - SL - DEBUG - 671 - "/app/email_handler.py:316" - replace_header_when_forward() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Delete Cc header, old value None
2026-07-13 17:54:36,179 - SL - DEBUG - 671 - "/app/email_handler.py:313" - replace_header_when_forward() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Replace To header, old: e1@sl.local, new: e1@sl.local
2026-07-13 17:54:36,179 - SL - INFO - 671 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Email has no unsubscribe header
2026-07-13 17:54:36,179 - SL - DEBUG - 671 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Forward mail from hey@google.com to john@wick.com, mail_options:[], rcpt_options:[] 
2026-07-13 17:54:36,181 - SL - DEBUG - 671 - "/app/app/mail_sender.py:156" - _send_to_smtp() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - getting a smtp connection takes seconds 0.0013813972473144531
2026-07-13 17:54:36,181 - SL - DEBUG - 671 - "/app/app/mail_sender.py:163" - _send_to_smtp() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Sendmail mail_from:sl.lmycyibsfqqdemzygi4dgnc5.e6al3q2pm6lsa@sl.local, rcpt_to:john@wick.com, header_from:"hey at google.com" <hey_at_google_com_pfxrzq@sl.local>, header_to:e1@sl.local, header_cc:None
2026-07-13 17:54:36,184 - SL - INFO - 671 - "/app/email_handler.py:2367" - _handle() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.2055816650390625 seconds with return code '250 Message accepted for delivery'<<===
```

**Success log lines that answer Q1 [OBSERVED]:**

| Log text (verbatim) | Level | Emitted at (observed `file:line`) |
|---------------------|-------|-----------------------------------|
| `Forward phase hey@google.com(hey@google.com) -> e1@sl.local` | DEBUG | `email_handler.py:2202` |
| `Create <EmailLog 2> for <Contact 2 hey@google.com 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>` | DEBUG | `email_handler.py:740` |
| `getting a smtp connection takes seconds 0.0013813972473144531` | DEBUG | `app/mail_sender.py:156` |
| `Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.2055816650390625 seconds with return code '250 Message accepted for delivery'<<===` | INFO | `email_handler.py:2367` |

The terminating status string `250 Message accepted for delivery` is the constant `E200` (`app/email/status.py:2`) **[INFERRED from source; the string itself is OBSERVED in the summary line above]**. The `getting a smtp connection takes seconds` line (`app/mail_sender.py:156`) is emitted **unconditionally on the real send path** after the SMTP connection is obtained — it appears because `NOT_SEND_EMAIL=False` routes execution through `_send_to_smtp` (`app/mail_sender.py:144`) rather than the `NOT_SEND_EMAIL` short-circuit (`app/mail_sender.py:130-136`).

### 4.2 Non-existent alias — complete handler stdout and no-egress proof

**Pre-send absence check [OBSERVED]** — the target alias does not exist before the send:

```bash
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "select count(*) from alias where email='"'"'doesnotexist@sl.local'"'"';"'
```

```
0
```

Command (host) — same fixed payload, only the RCPT changes to a non-existent alias:

```bash
swaks --to doesnotexist@sl.local --from hey@google.com --server 127.0.0.1:20381 --data - < /tmp/payload.eml
```

Complete SMTP transcript **[OBSERVED]** — terminal status is `550 SL E515 Email not exist` (note `<**`, an SMTP error reply):

```
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 0e8a6464d539 Python SMTP 1.4.2
 -> EHLO 0e8a6464d539
<-  250-0e8a6464d539
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<hey@google.com>
<-  250 OK
 -> RCPT TO:<doesnotexist@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Mon, 13 Jul 2026 12:00:00 +0000
 -> To: e1@sl.local
 -> From: hey@google.com
 -> Subject: Blitzy fixed forward probe
 -> Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
 -> Content-Type: text/plain; charset=UTF-8
 -> Content-Transfer-Encoding: 7bit
 -> 
 -> Fixed byte-identical body for run-to-run replay. Do not change.
 -> 
 -> .
<** 550 SL E515 Email not exist
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

Complete handler stdout for the failure **[OBSERVED]** (log-tracing id `9497ee13-4cc3-4088-93aa-7b15eae78ae3`):

```
2026-07-13 17:56:49,490 - SL - DEBUG - 671 - "/app/app/log.py:24" - set_message_id() - a6fdff0a-4870-4c40-b036-e57944e634c4 - set message_id 9497ee13-4cc3-4088-93aa-7b15eae78ae3
2026-07-13 17:56:49,490 - SL - DEBUG - 671 - "/app/email_handler.py:2342" - _handle() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - ====>=====>====>====>====>====>====>====>
2026-07-13 17:56:49,490 - SL - INFO - 671 - "/app/email_handler.py:2343" - _handle() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - New message, mail from hey@google.com, rctp tos ['doesnotexist@sl.local'] 
2026-07-13 17:56:49,491 - SL - DEBUG - 671 - "/app/email_handler.py:1963" - handle() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - Cannot parse Postfix queue ID from None None
2026-07-13 17:56:49,493 - SL - DEBUG - 671 - "/app/email_handler.py:1980" - handle() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - ==>> Handle mail_from:hey@google.com, rcpt_tos:['doesnotexist@sl.local'], header_from:hey@google.com, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<blitzy-fixed-probe-2cd6ee777f8c@test.local>, client_ip:None, headers:[('Date', 'Mon, 13 Jul 2026 12:00:00 +0000'), ('To', 'e1@sl.local'), ('From', 'hey@google.com'), ('Subject', 'Blitzy fixed forward probe'), ('Message-ID', '<blitzy-fixed-probe-2cd6ee777f8c@test.local>'), ('Content-Type', 'text/plain; charset=UTF-8'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 17:56:49,496 - SL - DEBUG - 671 - "/app/email_handler.py:2202" - handle() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - Forward phase hey@google.com(hey@google.com) -> doesnotexist@sl.local
2026-07-13 17:56:49,502 - SL - DEBUG - 671 - "/app/email_handler.py:545" - handle_forward() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - alias doesnotexist@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 17:56:49,508 - SL - INFO - 671 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - Cannot auto-create custom domain alias for doesnotexist@sl.local because there's no custom domain for sl.local
2026-07-13 17:56:49,508 - SL - INFO - 671 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - Cannot auto-create doesnotexist@sl.local since it has no directory separator
2026-07-13 17:56:49,508 - SL - DEBUG - 671 - "/app/email_handler.py:551" - handle_forward() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - alias doesnotexist@sl.local cannot be created on-the-fly, return 550
2026-07-13 17:56:49,509 - SL - INFO - 671 - "/app/email_handler.py:2367" - _handle() - 9497ee13-4cc3-4088-93aa-7b15eae78ae3 - Finish mail_from hey@google.com, rcpt_tos ['doesnotexist@sl.local'], takes 0.019599199295043945 seconds with return code '550 SL E515 Email not exist'<<===
```

**Non-existent-alias log lines that answer Q1 [OBSERVED]:**

| Log text (verbatim) | Level | Emitted at (observed `file:line`) |
|---------------------|-------|-----------------------------------|
| `alias doesnotexist@sl.local not exist. Try to see if it can be created on the fly` | DEBUG | `email_handler.py:545` |
| `Cannot auto-create custom domain alias for doesnotexist@sl.local because there's no custom domain for sl.local` | INFO | `app/alias_utils.py:104` |
| `Cannot auto-create doesnotexist@sl.local since it has no directory separator` | INFO | `app/alias_utils.py:165` |
| `alias doesnotexist@sl.local cannot be created on-the-fly, return 550` | DEBUG | `email_handler.py:551` |
| `Finish mail_from hey@google.com, rcpt_tos ['doesnotexist@sl.local'], takes 0.019599199295043945 seconds with return code '550 SL E515 Email not exist'<<===` | INFO | `email_handler.py:2367` |

The terminating status string `550 SL E515 Email not exist` is the constant `E515` (`app/email/status.py:51`) **[INFERRED from source; the string itself is OBSERVED above]**. The `return 550` branch returns `[(False, status.E515)]` (`email_handler.py:554-555`) **[INFERRED]**.

**No-egress / no-side-effects proof [OBSERVED].** After the failure, all four forward-related tables are unchanged and the sink stored nothing new. Command and output:

```bash
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select '"'"'contact'"'"', count(*) from contact
union all select '"'"'email_log'"'"', count(*) from email_log
union all select '"'"'user_audit_log'"'"', count(*) from user_audit_log
union all select '"'"'message_id_matching'"'"', count(*) from message_id_matching;"'
docker exec "$CID" bash -lc 'ls /tmp/blitzy_inv_sink_capture | wc -l'
```

```
contact|2
email_log|4
user_audit_log|1
message_id_matching|0
3
```

The counts `(contact 2, email_log 4, user_audit_log 1, message_id_matching 0)` are **identical to the post-run-#3 state** (Section 7.1), and the sink still holds only the **3** successful forwards — the failed send created **no database rows and emitted no downstream message**.

### 4.3 Correlation-id (log-tracing id) semantics — corrected, with live proof

`set_message_id()` logs **before** it assigns the module global: `LOG.d("set message_id %s", message_id)` at `app/log.py:24` runs before `_MESSAGE_ID = message_id` at `app/log.py:25`. The formatter's filter runs at emit time (`EmailHandlerFilter.filter`, `app/log.py:31-34`) and reads `_MESSAGE_ID` via `get_message_id()` (`app/log.py:36-37`), so the **setter line itself carries the *previous* id** (blank on the very first message), and the **new** id appears from the **following** line onward. This was confirmed live four times **[OBSERVED]**:

| Event | Setter line prefix (prior id) | New id set on next line |
|-------|-------------------------------|--------------------------|
| run #1 (first message) | *(blank)* | `83d50cd9-5687-4f5c-b787-025bbabb0b12` |
| run #2 | `83d50cd9-5687-4f5c-b787-025bbabb0b12` | `4107e38b-d0a5-41c1-969b-52e40f065d28` |
| run #3 | `4107e38b-d0a5-41c1-969b-52e40f065d28` | `a6fdff0a-4870-4c40-b036-e57944e634c4` |
| failure | `a6fdff0a-4870-4c40-b036-e57944e634c4` | `9497ee13-4cc3-4088-93aa-7b15eae78ae3` |

(See the first line of each handler block in Sections 4.1 and 4.2.) This log-tracing id is `str(uuid.uuid4())` created in `_handle()` (`email_handler.py:2339`), passed to `set_message_id()` (`email_handler.py:2340`); when `handle()` runs under Postfix it is instead the queue id (`email_handler.py:1959`), which here logs `Cannot parse Postfix queue ID from None None` (`email_handler.py:1963`) because `swaks` supplies no queue id. This id is **not** an email `Message-ID` (Q2).


---

## 5. Q2 — SL Message-ID vs. original Message-ID

The question conflates what the running code treats as **three distinct identifiers**. The investigation observed each and disambiguates them.

### 5.1 The three identifiers

| # | Identifier | Where it comes from | Observed on this forward? |
|---|-----------|---------------------|----------------------------|
| 1 | **Original `Message-ID` header** — `<blitzy-fixed-probe-2cd6ee777f8c@test.local>` | Set by the sender; **preserved** by the forward path (`email_handler.py:799` comment `# do not delete original message id`; the header is kept via `headers_to_keep`, `email_handler.py:793`). | **[OBSERVED]** preserved unchanged in the delivered message. |
| 2 | **Log-tracing id** — a per-message `uuid4` (run #1 value `83d50cd9-5687-4f5c-b787-025bbabb0b12`) | `str(uuid.uuid4())` in `_handle()` (`email_handler.py:2339`), injected into every log line as `%(message_id)s` (`app/log.py:14`). | **[OBSERVED]** varies every message (Section 4.3). Not an email header. |
| 3 | **`sl_message_id`** — a `make_msgid(...)` value | Built in the **reply** path `replace_original_message_id()` (`email_handler.py:1311`), persisted in `message_id_matching` (`MessageIDMatching`, `app/models.py:3365`, column `sl_message_id` `app/models.py:3371`) and `email_log.sl_message_id` (`app/models.py:2116`). | **[OBSERVED] NOT created by the forward** — see 5.3. |

### 5.2 The delivered `Message-ID` equals the sent one (byte-for-byte)

The sink stored the delivered `.eml` for each of the three identical runs. The `Message-ID` line is identical to the sent one in every run **[OBSERVED]**:

```bash
for f in msg_0001.eml msg_0002.eml msg_0003.eml; do
  echo "--- $f ---"; grep -E '^(Message-ID|From|X-SimpleLogin-EmailLog-ID):' "$f"
done
```

```
--- msg_0001.eml ---
Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
X-SimpleLogin-EmailLog-ID: 2
From: "hey at google.com" <hey_at_google_com_pfxrzq@sl.local>
--- msg_0002.eml ---
Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
X-SimpleLogin-EmailLog-ID: 3
From: "hey at google.com" <hey_at_google_com_pfxrzq@sl.local>
--- msg_0003.eml ---
Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
X-SimpleLogin-EmailLog-ID: 4
From: "hey at google.com" <hey_at_google_com_pfxrzq@sl.local>
```

Sent `Message-ID` = `<blitzy-fixed-probe-2cd6ee777f8c@test.local>` (Section 3); delivered `Message-ID` = the same, for all three runs. **Difference = zero bytes.** Direct proof the forward path does **not** rewrite the `Message-ID` **[OBSERVED]**. (The forward path only rewrites threading headers `In-Reply-To`/`References` via `replace_sl_message_id_by_original_message_id()`, defined `email_handler.py:931`, called `email_handler.py:860` — it does **not** touch `Message-ID` **[INFERRED]**.)

The delivered messages also confirm there is **exactly one** `Message-ID` header (and exactly one `From`), removing the earlier ambiguity of a duplicated field. Full delivered header block for run #1 **[OBSERVED]**:

```
Date: Mon, 13 Jul 2026 12:00:00 +0000
Subject: Blitzy fixed forward probe
Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 7bit
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 2
X-SimpleLogin-Envelope-From: hey@google.com
X-SimpleLogin-Original-From: hey@google.com
X-SimpleLogin-Envelope-To: e1@sl.local
From: "hey at google.com" <hey_at_google_com_pfxrzq@sl.local>
To: e1@sl.local

Fixed byte-identical body for run-to-run replay. Do not change.
```

### 5.3 `sl_message_id` is a reply-phase artifact — not written by any forward

Labeled snapshots of `message_id_matching` **and** `email_log.sl_message_id` were taken before and after each transaction. `message_id_matching` stayed empty throughout, and `email_log.sl_message_id` is empty for every forwarded row **[OBSERVED]**:

```bash
# taken at each labeled checkpoint
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select '"'"'message_id_matching_count'"'"', count(*) from message_id_matching;"'
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, coalesce(message_id,'"'"'<empty>'"'"'), coalesce(sl_message_id,'"'"'<empty>'"'"') from email_log order by id;"'
```

| Checkpoint (label) | `message_id_matching` count | `email_log` rows: `id / message_id / sl_message_id` |
|--------------------|------------------------------|------------------------------------------------------|
| BEFORE any send | `0` | `1 / <empty> / <empty>` (pre-seeded baseline) |
| AFTER run #1 | `0` | `1 / <empty> / <empty>` ; `2 / <blitzy-fixed-probe-2cd6ee777f8c@test.local> / <empty>` |
| AFTER run #2 | `0` | + `3 / <blitzy-fixed-probe-2cd6ee777f8c@test.local> / <empty>` |
| AFTER run #3 | `0` | + `4 / <blitzy-fixed-probe-2cd6ee777f8c@test.local> / <empty>` |
| AFTER failure | `0` | (unchanged: rows 1–4) |

**[OBSERVED]** `message_id_matching` is `0` at every checkpoint; `email_log.sl_message_id` is empty on all forward rows; `email_log.message_id` holds the preserved original id. **[INFERRED]** the `sl_message_id`/`make_msgid` machinery (`email_handler.py:1311`, `MessageIDMatching.create` `email_handler.py:1316`, `del msg[MESSAGE_ID]` then reassign `email_handler.py:1338`) executes only on the **reply** path (`replace_original_message_id()` called at `email_handler.py:1202`), which this forward-only investigation did not trigger.

### 5.4 Q2 answer and bounded conclusion

- **On a forward**, there is **no** distinct "SL Message-ID" applied to the message: the original `Message-ID` is preserved verbatim (difference = 0 bytes) **[OBSERVED]**.
- The `sl_message_id` that `make_msgid(str(email_log.id), get_email_domain_part(alias.email))` produces is a **reply-phase** identifier, stored in `message_id_matching`, and was not produced by any forward here **[OBSERVED that it is absent; INFERRED where it is produced]**.
- **Bounded conclusion / hypothesis:** the perception that the "SL Message-ID differs from the original" is consistent with the **phase-dependent** handling — preserved on forward, replaced on reply. Because the reply path was not exercised, this remains a **hypothesis**, not a demonstrated defect.


---

## 6. Q3 — Transformed `From` header and reverse-alias (reply-email) format

### 6.1 The transformed `From` header (delivered)

The delivered `From` header, read verbatim from the sink `.eml` **[OBSERVED]**:

```
From: "hey at google.com" <hey_at_google_com_pfxrzq@sl.local>
```

This is identical across all three byte-identical runs (Section 5.2). The handler also logged the replacement inline **[OBSERVED]**:

```
2026-07-13 17:54:36,179 - SL - DEBUG - 671 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 83d50cd9-5687-4f5c-b787-025bbabb0b12 - From header, new:"hey at google.com" <hey_at_google_com_pfxrzq@sl.local>, old:hey@google.com
```

The new `From` value is produced by `contact.new_addr()` and assigned as `new_from_header` (`email_handler.py:865`), then logged at `email_handler.py:867` **[INFERRED for the code path; the value is OBSERVED]**.

### 6.2 Anatomy of the value

- **Display name** `"hey at google.com"` — the sender address `hey@google.com` with `@` → ` at `. This is `SenderFormatEnum.AT` (value `0`, the default; `app/models.py:2028-2029`), applied in `Contact.new_addr()` (`app/models.py:2008`) **[OBSERVED value; INFERRED mechanism]**.
- **Angle-addr** `<hey_at_google_com_pfxrzq@sl.local>` — the reverse-alias stored in `Contact.reply_email` (`app/models.py:1899`). Local part = `{sanitized_sender}_{random}`:
  - `hey_at_google_com` — the sender with non-alphanumerics collapsed to `_`.
  - `pfxrzq` — a random suffix (here 6 chars).
  - No `ra+`/`reply+` prefix — that legacy prefix is **commented out** in this version (`app/email_utils.py:1141`, `:1147`) **[INFERRED from source]**.
  - Domain `sl.local` = `EMAIL_DOMAIN` (Section 2.6).

### 6.3 Reverse-alias generation branch and length distribution

The seeded user `john@wick.com` has `include_sender_in_reverse_alias = True` (Section 2.7), so `generate_reply_email()` (`app/email_utils.py:1103`) takes the **sender-included** branch (`app/email_utils.py:1116-1117`), whose random suffix is `random_string(randint(5, 10))` (`app/email_utils.py:1138`, f-string `:1139-1143`) **[INFERRED mechanism]**. To observe the actual distribution, three additional **new senders** were forwarded to `e1@sl.local` (sink messages `#4–#6`), and the reverse-alias local-part lengths measured **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, website_email, reply_email, length(split_part(reply_email,'"'"'@'"'"',1)) as local_len
from contact where alias_id = 5 order by id;"'
```

```
2|hey@google.com|hey_at_google_com_pfxrzq@sl.local|24
3|alice@example.org|alice_at_example_org_pzfsu@sl.local|26
4|bob@example.org|bob_at_example_org_zurehw@sl.local|25
5|carol@example.org|carol_at_example_org_hkntdaxls@sl.local|30
```

Random suffix lengths (local-part after the final `_`): `pfxrzq`=6, `pzfsu`=5, `zurehw`=6, `hkntdaxls`=9 — all within `randint(5, 10)` **[OBSERVED]**. The full local-part length varies with the sender prefix, so it is **not** fixed; only the *format* is stable.

### 6.4 Cross-check: the transformed address is a valid reverse-alias

The delivered `From`'s angle-addr round-trips through the product's own predicate and resolves back to the contact **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/python -c "
from app.email_utils import is_reverse_alias
from app.models import Contact
print(\"is_reverse_alias:\", is_reverse_alias(\"hey_at_google_com_pfxrzq@sl.local\"))
c = Contact.get_by(reply_email=\"hey_at_google_com_pfxrzq@sl.local\")
print(\"resolves to Contact id/alias_id/website_email:\", c.id, c.alias_id, c.website_email)
"'
```

```
is_reverse_alias: True
resolves to Contact id/alias_id/website_email: 2 5 hey@google.com
```

`is_reverse_alias(...)` (`app/email_utils.py:1156`) returns `True`, and the address maps back to `Contact` id 2 on alias 5 for sender `hey@google.com` — i.e. the delivered `From` is the reverse-alias for this (sender, alias) pair.

---

## 7. Q4 — Database records created by a single forward

### 7.1 Before / after snapshots (full projections matching displayed columns)

Counts before any send, and row-level detail after each of the three identical runs, using SQL projections whose selected columns match the displayed columns exactly.

**BEFORE any send [OBSERVED]:**

```bash
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select '"'"'contact'"'"', count(*) from contact
union all select '"'"'email_log'"'"', count(*) from email_log
union all select '"'"'user_audit_log'"'"', count(*) from user_audit_log
union all select '"'"'message_id_matching'"'"', count(*) from message_id_matching;"'
```

```
contact|1
email_log|1
user_audit_log|0
message_id_matching|0
```

**AFTER run #1 (first send from a new sender) — three new rows [OBSERVED]:**

```bash
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, alias_id, website_email, reply_email, created_at from contact where id=2;"'
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, user_id, action, message, created_at from user_audit_log where id=1;"'
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, contact_id, user_id, mailbox_id, coalesce(message_id,'"'"'<empty>'"'"'), coalesce(sl_message_id,'"'"'<empty>'"'"'), created_at from email_log where id=2;"'
```

```
2|5|hey@google.com|hey_at_google_com_pfxrzq@sl.local|2026-07-13 17:54:36.152853
1|1|create_contact|Created contact 2 (hey@google.com)|2026-07-13 17:54:36.159224
2|2|1|1|<blitzy-fixed-probe-2cd6ee777f8c@test.local>|<empty>|2026-07-13 17:54:36.171573
```

**AFTER run #2 and run #3 (repeat sends, same sender) — only a new `EmailLog` each [OBSERVED]:**

```bash
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, contact_id, user_id, mailbox_id, coalesce(message_id,'"'"'<empty>'"'"'), coalesce(sl_message_id,'"'"'<empty>'"'"'), created_at from email_log where id in (3,4) order by id;"'
docker exec "$CID" bash -lc 'export PGPASSWORD=mypassword; psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select '"'"'contact'"'"', count(*) from contact union all select '"'"'email_log'"'"', count(*) from email_log union all select '"'"'user_audit_log'"'"', count(*) from user_audit_log union all select '"'"'message_id_matching'"'"', count(*) from message_id_matching;"'
```

```
3|2|1|1|<blitzy-fixed-probe-2cd6ee777f8c@test.local>|<empty>|2026-07-13 17:55:17.34962
4|2|1|1|<blitzy-fixed-probe-2cd6ee777f8c@test.local>|<empty>|2026-07-13 17:55:50.917626
contact|2
email_log|4
user_audit_log|1
message_id_matching|0
```

### 7.2 Records created by one forward — summary

| Send | Records created | IDs | `created_at` (observed) |
|------|-----------------|-----|--------------------------|
| **Run #1** (new sender) | `Contact`, `UserAuditLog`, `EmailLog` | 2, 1, 2 | `17:54:36.152853`, `17:54:36.159224`, `17:54:36.171573` |
| **Run #2** (repeat) | `EmailLog` only | 3 | `17:55:17.34962` |
| **Run #3** (repeat) | `EmailLog` only | 4 | `17:55:50.917626` |

Creation order within run #1 (by timestamp) **[OBSERVED]**: `Contact` (`.152853`) → `UserAuditLog` (`.159224`) → `EmailLog` (`.171573`). This matches the code order **[INFERRED]**: `create_contact()` creates the `Contact` (`app/contact_utils.py:91-102`) and emits the `UserAuditLog` (`app/contact_utils.py:104-109`), after which `forward_email_to_mailbox()` creates the `EmailLog` (`email_handler.py:732`, logged `email_handler.py:740`).

### 7.3 Why a repeat send reuses the `Contact` — app lookup vs. DB constraint (distinct)

Two independent mechanisms enforce one `Contact` per (alias, sender); they are **not** the same line and the report cites both:

- **Application-level reuse lookup [INFERRED]:** `create_contact()` first calls `Contact.get_by(alias_id=alias.id, website_email=email)` (`app/contact_utils.py:85`); on a hit it returns the existing contact instead of inserting — this is why runs #2/#3 add no `Contact` row.
- **Database-level uniqueness constraint [INFERRED]:** the `Contact` model declares `sa.UniqueConstraint("alias_id", "website_email", name="uq_contact")` (`app/models.py:1874-1876`, literal at `:1875`), created by migration `2020_031711_0809266d08ca_.py:45` (`op.create_unique_constraint("uq_contact", "contact", ["alias_id", "website_email"])`). This is the schema-enforced guarantee, independent of the app lookup.

### 7.4 ID and timestamp provenance

Every row's `id` is the autoincrement PK and `created_at` is `arrow.utcnow()`, both from `ModelMixin` (`app/models.py:62`; `id` `:63`; `created_at` `:64`) **[INFERRED mechanism; the concrete ids/timestamps above are OBSERVED]**. Consequently the `id` values and `created_at` timestamps are **run-to-run variable** by construction (Section 8).


---

## 8. Run-to-run stability (directly addressing "inconsistent behavior")

The **same 314-byte payload** (`sha256=c1dd2e00b7fc4009815f7b4a13b61151c367744ec480c5cfa2e599daf1f84471`) was replayed three times; the handler received byte-identical input each time (`inbound-line-sha256=ec07697b1bb5eda1819cc7d42ac66bed8757f892a0bc36fcb7a613d42e96dabc`, identical across all three runs — Section 3). Observed distribution across the three runs:

| Field | Run #1 | Run #2 | Run #3 | Classification |
|-------|--------|--------|--------|----------------|
| SMTP terminal status | `250 Message accepted for delivery` | same | same | **STABLE** [OBSERVED] |
| Success summary template (`email_handler.py:2367`) | present | present | present | **STABLE** [OBSERVED] |
| Delivered `Message-ID` | `<blitzy-fixed-probe-2cd6ee777f8c@test.local>` | same | same | **STABLE** [OBSERVED] |
| Delivered `From` format | `"hey at google.com" <hey_at_google_com_pfxrzq@sl.local>` | same | same | **STABLE** [OBSERVED] |
| `Contact.reply_email` (contact reused) | `hey_at_google_com_pfxrzq@sl.local` | same | same | **STABLE** [OBSERVED] |
| `EmailLog.id` | `2` | `3` | `4` | **VARIABLE** [OBSERVED] |
| `EmailLog.created_at` | `2026-07-13 17:54:36.171573` | `2026-07-13 17:55:17.34962` | `2026-07-13 17:55:50.917626` | **VARIABLE** [OBSERVED] |
| Log-tracing `uuid4` | `83d50cd9-5687-4f5c-b787-025bbabb0b12` | `4107e38b-d0a5-41c1-969b-52e40f065d28` | `a6fdff0a-4870-4c40-b036-e57944e634c4` | **VARIABLE** [OBSERVED] |
| Delivered envelope-from (VERP, `mail_sender.py:163`) | `sl.lmycyibsfqqdemzygi4dgnc5.e6al3q2pm6lsa@sl.local` | `sl.lmycyibtfqqdemzygi4dgnk5.vyb6ku3x73gks@sl.local` | `sl.lmycyibufqqdemzygi4dgnk5.lb2s6zhltoi2c@sl.local` | **VARIABLE** [OBSERVED] |
| `X-SimpleLogin-EmailLog-ID` header | `2` | `3` | `4` | **VARIABLE** [OBSERVED] |
| Full delivered `.eml` sha256 | `caaac0e1aac3b9085de975a458197a50a24a43fce4bfabe1951da3102e1e5dd8` | `62330b65e531b0b8b8260d40b49fc056838e8d9b3ba0cfd78ae0d1d24e64f6db` | `ca5690f49f1a4f795b3ac79591f733aa83eb3261dde7ca15ff67b9628d2f3a58` | **VARIABLE** [OBSERVED] |

**Nuance worth highlighting [OBSERVED]:** even with byte-identical *input*, the full delivered `.eml` bytes differ every run, because the message embeds the fresh `EmailLog.id` in the `X-SimpleLogin-EmailLog-ID` header (`2` → `3` → `4` above). A `diff` of two consecutive delivered `.eml` files confirms that header is the **only** line that changes (`X-SimpleLogin-EmailLog-ID: 2` vs `3`). So "the forwarded email is not byte-identical run to run" is **true and expected** — while the two fields a recipient usually notices (`Message-ID`, `From` format) are stable. The message is **not** DKIM-signed under this default configuration: `DKIM_PRIVATE_KEY_PATH` is commented out (`example.env:72`), so `config.DKIM_PRIVATE_KEY` is `None` (`app/config.py:184`) and the unconditional `add_dkim_signature(msg, EMAIL_DOMAIN)` call (`email_handler.py:891`) is a no-op via the `if config.DKIM_PRIVATE_KEY:` guard (`app/email_utils.py:490`) — consistent with the Section 5.2 delivered header set, which contains no `DKIM-Signature`.

**Bounded conclusion / hypothesis (finding-aware):**

- **[OBSERVED]** On the **forward** path, with identical input, behavior is *format-stable* (templates, status codes, preserved `Message-ID`, `From`/reverse-alias format) and *value-variable* (`EmailLog.id`, timestamps, log-tracing uuid, VERP envelope-from, whole-message bytes — solely due to the `X-SimpleLogin-EmailLog-ID` increment; the message is not DKIM-signed by default).
- **[HYPOTHESIS]** A user perceiving "inconsistent behavior" is most plausibly seeing (i) these intentionally per-message-variable fields and/or (ii) **phase-dependent** `Message-ID` handling (preserved on forward vs. replaced on reply, Section 5).
- This report deliberately does **not** claim the reported production issue is "not a defect" or globally deterministic: the **reply** and **bounce** paths were not exercised here, so any statement about them would be unsupported. The scope of these conclusions is exactly the **forward transactions observed above**.

---

## 9. Official SimpleLogin context (product-level), with local-authoritative caveat

These official sources frame the reverse-alias and `From`-replacement concepts at the product level. **The running local code (cited throughout) remains the source of truth for exact values**; the public docs describe the hosted `.co` service and differ from the local runtime in the ways noted.

- **Reverse-alias overview** — SimpleLogin docs: [https://simplelogin.io/docs/getting-started/reverse-alias/](https://simplelogin.io/docs/getting-started/reverse-alias/). A reverse-alias is unique per (sender, alias) and by default composed of random characters — corroborating Q3's mechanism.
- **FAQ** — [https://simplelogin.io/faq/](https://simplelogin.io/faq/). Describes the reverse-alias as a special alias created per alias-and-contact that lets you send from your alias, i.e. the `From` a recipient sees is the alias/reverse-alias, not the real mailbox — corroborating the `From` replacement (`contact.new_addr()`).
- **Sending from an alias** — [https://simplelogin.io/docs/getting-started/send-email/](https://simplelogin.io/docs/getting-started/send-email/). Confirms replies target the reverse-alias (the reply-phase relevant to identifier #3 in Section 5).
- **Reverse-alias generation algorithm** — SimpleLogin blog: [https://simplelogin.io/blog/reverse-alias/](https://simplelogin.io/blog/reverse-alias/). Documents that reverse-aliases historically looked like `ra+<random>@simplelogin.co` and that a change later *included the sender address* in the reverse-alias (form `ra+sender.at.domain.com+random_string@simplelogin.co`). This directly explains the two format families in the code (the sender-included branch vs. the default branch in `generate_reply_email`).

**Local-runtime-authoritative caveat [OBSERVED vs. public docs]:** in this local build the domain is `sl.local` (not `simplelogin.co`); the `ra+` prefix is **commented out** (`app/email_utils.py:1141`, `:1147`), so the observed local reverse-alias is *prefix-less* `hey_at_google_com_pfxrzq@sl.local`; and the *sender-included* branch is active because the seeded user has `include_sender_in_reverse_alias = True`. Where public docs and local behavior differ, the **observed local values above govern**.

---

## 10. Coverage pass

| Question | Named item | Where answered | Label |
|----------|-----------|----------------|-------|
| Q1 | Success log text + level | §4.1 (INFO `Finish` summary ending `'250 Message accepted for delivery'<<===`, `email_handler.py:2367`; `E200` `app/email/status.py:2`) | [OBSERVED] |
| Q1 | Non-existent-alias log text + level | §4.2 (DEBUG `not exist` `:545`; DEBUG `cannot be created on-the-fly, return 550` `:551`; INFO `Finish` summary ending `'550 SL E515 Email not exist'<<===` `:2367`; `E515` `app/email/status.py:51`) | [OBSERVED] |
| Q2 | The "SL Message-ID" generated during forwarding | §5.2–5.4 (none applied on forward; original preserved, 0-byte diff) | [OBSERVED] |
| Q2 | How it differs from the original | §5.2 (identical — zero difference on forward); §5.1/5.3 (`sl_message_id` is a reply-phase artifact) | [OBSERVED]+[INFERRED] |
| Q3 | Exact transformed `From` header | §6.1 (`"hey at google.com" <hey_at_google_com_pfxrzq@sl.local>`) | [OBSERVED] |
| Q3 | Reverse-alias (reply-email) format | §6.2–6.4 (`{sanitized_sender}_{random 5–10}@sl.local`, prefix-less; `is_reverse_alias`=True) | [OBSERVED]+[INFERRED] |
| Q4 | Every record created by one forward | §7.1–7.2 (first: `Contact` 2 + `UserAuditLog` 1 + `EmailLog` 2; repeat: `EmailLog` only) | [OBSERVED] |
| Q4 | Actual record IDs and timestamps | §7.1 (Contact id 2, UserAuditLog id 1, EmailLog ids 2/3/4; timestamps from `2026-07-13 17:54:36.152853` through `2026-07-13 17:55:50.917626`) | [OBSERVED] |
| "inconsistent behavior" | Reproduce, don't stabilize | §3 (byte-identical replay), §8 (observed distribution, bounded hypothesis) | [OBSERVED]+[HYPOTHESIS] |


---

## 11. Cleanup & teardown (read-only guarantee restored)

The investigation was read-only against the SimpleLogin source and used only temporary, uniquely-named artifacts. All of them — the investigation container, its services (Postgres/Redis/sink, which live only inside the container), the host observation directory, and a stale gitignored `.env` left in the host working tree by an earlier run — were removed. Teardown targeted **only** validated, uniquely-named resources by their captured identifiers (mitigating path-traversal / wrong-target risk, CWE-22): the container was removed by its **exact captured id** after confirming it still carried the investigation label, and each filesystem path was re-resolved with `realpath` and prefix-checked before removal.

**Pre-cleanup safety validation [OBSERVED]:**

```bash
CID="$(cat /tmp/blitzy_inv_06d10137/container_id.txt)"
docker ps -a --filter "id=${CID}" --format '{{.ID}}  {{.Label "blitzy-investigation"}}  {{.Image}}  {{.Status}}'
git check-ignore -v .env         # confirm host .env is gitignored
git ls-files --error-unmatch .env  # confirm host .env is NOT tracked
```

```
0e8a6464d539  app_2cd6ee777f8c  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0  Up 33 minutes
.gitignore:4:.env	.env
error: pathspec '.env' did not match any file(s) known to git
```

The container id matches the captured value and carries the `blitzy-investigation=app_2cd6ee777f8c` label; the host `.env` is gitignored (`.gitignore:4`) and untracked — safe to remove.

**Teardown [OBSERVED]** (each step with its exit status):

```bash
docker rm -f "$CID"; echo "exit=$?"
rm -rf /tmp/blitzy_inv_06d10137; echo "exit=$?"     # realpath-validated under /tmp
rm -f  "$REPO/.env";           echo "exit=$?"        # gitignored + untracked
```

```
0e8a6464d53930be6167ac1e53a39087111e2531212e38dafa4cfb591c076967
exit=0
exit=0
exit=0
```

**Post-cleanup verification [OBSERVED]:**

```bash
git status --porcelain
git status --short --ignored | grep -E '\.env' || echo "(.env no longer present — clean)"
docker ps -a --filter "id=${CID}" --format '{{.ID}} {{.Status}}' | grep . || echo "(container absent — removed)"
docker ps -a --filter "label=blitzy-investigation=app_2cd6ee777f8c" --format '{{.ID}}' | grep . || echo "(no labelled investigation containers remain)"
[ -d /tmp/blitzy ] && echo "intact: /tmp/blitzy"; [ -d .git ] && echo "intact: repo/.git"
```

```
 M blitzy/documentation/app_2cd6ee777f8c.md
(.env no longer present — clean)
(container absent — removed)
(no labelled investigation containers remain)
intact: /tmp/blitzy
intact: repo/.git
```

**Extended resource-absence checks [OBSERVED]** (host-level; the investigation's services ran *inside* the container and published no host ports, volumes, or networks):

```bash
docker volume ls  --filter "label=blitzy-investigation=app_2cd6ee777f8c" --format '{{.Name}}' | grep . || echo "(no investigation-labelled volumes)"
docker network ls --filter "label=blitzy-investigation=app_2cd6ee777f8c" --format '{{.Name}}' | grep . || echo "(no investigation-labelled networks)"
for p in 20381 1025 15432 6379; do ss -ltn | grep -q ":$p " && echo "port $p: on host" || echo "port $p: not on host"; done
[ -e /tmp/blitzy_inv_06d10137 ] && echo "PRESENT" || echo "(/tmp/blitzy_inv_06d10137 absent)"
```

```
(no investigation-labelled volumes)
(no investigation-labelled networks)
port 20381: not on host
port 1025: not on host
port 15432: not on host
port 6379: not on host
(/tmp/blitzy_inv_06d10137 absent)
```

No investigation-labelled Docker **volume** or **network** was ever created (the container ran with neither `-v` nor a custom `--network`), and none of the service **ports** (`20381` handler, `1025` sink, `15432` Postgres, `6379` Redis) is bound on the host — confirming every service **process** lived inside the now-removed container. Together with the container / `.env` / temp-dir removals above, the investigation leaves **no** process, port, container, network, or volume behind.

The only change in the working tree is this document (`blitzy/documentation/app_2cd6ee777f8c.md`); the stale `.env` is gone; the investigation container and all label-matched containers are removed; and the shared workspace and repository git metadata are intact. The SimpleLogin product source is byte-for-byte unchanged.

> **Note on the commands above.** Every command block in this report is shown exactly as executed against the now-removed container (`$CID = 0e8a6464d539…`) and its temporary paths (`/tmp/payload.eml`, `/tmp/blitzy_inv_sink_capture/…`). They are preserved as the reproducible record of what was run; the container and those paths no longer exist after this teardown. Re-running the investigation from Section 2 reproduces the same environment.
