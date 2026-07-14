# SimpleLogin Email-Forward Investigation — Runtime Evidence Report

**Reported production issue:** *"inconsistent behavior when emails are forwarded through SimpleLogin aliases."*

This report answers four precise questions by **running an actual email-forward operation through the real SimpleLogin email handler and capturing runtime values** — not values inferred from reading the code alone. Every value below is grounded with the exact command that produced it, the command's complete output, and a `file:line` reference into the source at commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`, and is explicitly labeled **[OBSERVED]** (captured from a live run) or **[INFERRED]** (explained from source, not directly observed). Material command outputs are reproduced in full; the single deliberate exception is the 255-line Alembic transition sequence, whose banner and boundary transitions are shown as a labeled **[EXCERPT]** in §2.7 and whose complete form is provided verbatim in **Appendix A** (§12).

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
| Q2 | On a **forward**, the original `Message-ID` header is **preserved unchanged** — delivered `Message-ID` equals the sent one, byte-for-byte (identical across 3 identical runs). The distinct `sl_message_id` produced by `make_msgid(...)` (`email_handler.py:1311`) is a **reply-phase** artifact persisted in `message_id_matching` (`app/models.py:3365`); it was **not created by any forward** (table stayed empty; `email_log.sl_message_id` stayed empty). A third, separate identifier — the per-message log-tracing id — is a `uuid4` (`email_handler.py:2339`) that varies every message. | [OBSERVED] + [INFERRED] disambiguation |
| Q3 | Delivered `From` after transformation: `"hey at google.com" <hey_at_google_com_lvpak@sl.local>` — display name is the sender with `@`→` at ` (`SenderFormatEnum.AT`, `app/models.py:2028-2029`); the reverse-alias local part is `{sanitized_sender}_{random}` with a prefix-less random suffix of 5–10 chars (`generate_reply_email`, `app/email_utils.py:1138-1143`). | [OBSERVED] |
| Q4 | A **first** forward from a new sender creates exactly three rows: `Contact` (id 2), `UserAuditLog` (id 1, action `create_contact`), `EmailLog` (id 2). A **repeat** forward from the same sender reuses the `Contact` and creates only a new `EmailLog` (ids 3, then 4). `message_id_matching` is never written on forward. | [OBSERVED] |

**Bounded conclusion [OBSERVED for the forward path; HYPOTHESIS beyond it]:** across three byte-identical replays of the same input, the forward path produced **stable** log templates, SMTP status codes, the preserved `Message-ID`, and the `From`/reverse-alias **format**; it produced **variable** `EmailLog.id`, `created_at` timestamps, the log-tracing `uuid4`, and the delivered-message VERP envelope-from. The most likely explanation of a *perceived* "inconsistency" is (i) these per-run-variable fields and (ii) the **phase-dependent** Message-ID handling (preserved on forward vs. replaced on reply). This is offered as a **hypothesis**: the reply path and bounce path were not exercised in this investigation, so this report does **not** assert the reported issue is "not a defect" or globally deterministic — only that the **forward** path behaved as recorded here.

---

## 1. Evidence conventions & scope discipline

- **Canonical entry point only.** Every message is injected over real SMTP into the `aiosmtpd` controller that `email_handler.py` starts on `127.0.0.1:20381` (`email_handler.py:2383`, `:2386`, `:2403`), using `swaks`. No value in this report comes from a direct Python call to `handle_forward()`/`forward_email_to_mailbox()`.
- **Canonical downstream sink.** The downstream MTA is **MailHog** — the sink the Agent Action Plan and `CONTRIBUTING.md:184-221` name for local email observation — listening on `127.0.0.1:1025` with its capture API/UI on `127.0.0.1:1080` (§2.5). Delivered-message headers (`From`, `Message-ID`) are read back from MailHog's capture API, so the `From` and `Message-ID` values in §5/§6 are read from a **canonical** sink, not a bypassing stand-in.
- **Byte-identical replay.** To characterize run-to-run behavior the *same unchanged bytes* are replayed (a single fixed DATA payload with exactly one `Message-ID`); the report presents the observed distribution rather than a stabilized variant.
- **Output completeness & one cosmetic normalization.** Command outputs are shown in full inside fenced blocks (the sole labeled exception being the 255-line migration transition sequence — full text in Appendix A). The one cosmetic normalization applied to captured output is that **trailing whitespace is trimmed** so the deliverable passes `git diff --check`; no visible character of any logged message is altered (SimpleLogin emits a trailing space after some list interpolations such as `rctp tos [...] `; that space carries no evidentiary value for Q1–Q4 and its removal does not change any reported message text).
- **Read-only against the product.** No file in the SimpleLogin source tree was modified; this was verified with `git status`/`git diff` (§11). The only persistent artifact produced by this task is this document. All temporary scripts, the investigation container, and its services are torn down (§11).
- **Labels.** **[OBSERVED]** = captured from the live run whose command is shown; **[OBSERVED-in-isolation]** = captured from a live run of an isolated snippet rather than the full handler; **[INFERRED]** = explained from the source at the cited `file:line`, not directly observed at runtime.
- **Credentials.** The Postgres password is never printed. Every `psql` invocation derives it once into a shell variable from `example.env` — `PGPASS="$(grep -m1 '^DB_URI=' /app/example.env | sed -E 's#.*//[^:]+:([^@]+)@.*#\1#')"` — and passes it via `PGPASSWORD="$PGPASS"`; any `DB_URI` shown in output is redacted to `<redacted>`.
- **Reading the `psql` output.** Every database query uses `psql … -tAc "<SQL>"`: `-t` = tuples-only (no header/footer), `-A` = unaligned output, `-c` = run the single command and exit. In unaligned mode `psql` separates selected columns with the default field separator `|`, so a row such as `2|5|hey@google.com|hey_at_google_com_lvpak@sl.local|2026-07-13 23:16:25.397606` is exactly the selected columns in order, `|`-delimited. The selected columns in each query match the columns described alongside it.

---

## 2. Environment & reproducibility transcript

A single self-contained, ordered reproduction script that performs every step below (in the exact order required) is provided in **Appendix B** (§13); the subsections here narrate and evidence each step of that script.

### 2.1 Host vs. container boundary

The canonical runtime is the user-specified Docker image (Python 3.10 + Postgres + Redis). Docker runs on the **host**; **all** investigation steps run **inside the container** via `docker exec`, with the working directory `/app` (the SimpleLogin checkout). The deliverable document is written on the **host** repository working tree. The image entrypoint is `/bin/bash`, so the container is started with `--entrypoint bash <image> -lc "sleep infinity"` (exact command below) and driven with `docker exec`. The full container id is **recorded to `/tmp/blitzy_inv_06d10137/container_id.txt`** at creation time so later steps and teardown (§11) can target it by its exact id.

**Container provisioning (host command):**

```bash
mkdir -p /tmp/blitzy_inv_06d10137
INV_NAME="sl_inv_06d10137_$(date +%s)"
IMG="ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0"
CID=$(docker run -d --name "$INV_NAME" \
        --label blitzy-investigation=app_2cd6ee777f8c \
        --entrypoint bash "$IMG" -lc "sleep infinity")
echo "$CID" > /tmp/blitzy_inv_06d10137/container_id.txt   # full id, recorded for safe teardown
echo "$CID"
```

Observed container identity **[OBSERVED]**:

```
name: sl_inv_06d10137_1783984331
id:   8f131273a9f01305018f45e321a4c019dda61708780bd3822e42d05e3362ce89
label: blitzy-investigation=app_2cd6ee777f8c
```

All subsequent host commands read the id back with `CID="$(cat /tmp/blitzy_inv_06d10137/container_id.txt)"`.

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

### 2.3 Required one-time environment fixes: `pyre2` and `swaks`

Two one-time corrections are applied to the pristine image before the reproduction can run: the `pyre2`/`google-re2` swap below, and installation of the `swaks` SMTP injection client (used from §3 onward), which is **absent** from the image. Both are **environment/venv** corrections — no product file is modified.

#### 2.3.1 `pyre2` replaces `google-re2`

The image venv ships `google-re2` (`google-re2==1.1.20250805`), whose `re2` module lacks `DOTALL`/`IGNORECASE`. Observed directly **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc '/app/venv/bin/python -c "import re2; print(\"DOTALL:\", getattr(re2,\"DOTALL\",\"MISSING\")); print(\"IGNORECASE:\", getattr(re2,\"IGNORECASE\",\"MISSING\"))"'
```

```
DOTALL: MISSING
IGNORECASE: MISSING
```

The **canonical committed source** dereferences exactly those attributes at import time. Read verbatim from the untouched commit with `git show` (a read-only inspection — no working-tree file is changed) **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'cd /app && git show HEAD:app/spamassassin_utils.py | sed -n "8p;13p"'
```

```
import re2 as re
divider_pattern = re.compile(rb"^(.*?)\r?\n(.*?)\r?\n\r?\n", re.DOTALL)
```

So on a canonical checkout `app/spamassassin_utils.py:8` is `import re2 as re` and `:13` calls `re.compile(..., re.DOTALL)`. Combining the two observations, the failing expression `re.compile(..., re.DOTALL)` where `re` is `google-re2`'s `re2` module is reproduced **in isolation** (no product file touched) **[OBSERVED-in-isolation]**:

```bash
docker exec "$CID" bash -lc '/app/venv/bin/python -c "import re2 as re; re.compile(rb\"^(.*?)\r?\n(.*?)\r?\n\r?\n\", re.DOTALL)"'
```

```
Traceback (most recent call last):
  File "<string>", line 1, in <module>
AttributeError: module 're2' has no attribute 'DOTALL'
```

**[INFERRED]** Because the module-level statement at `app/spamassassin_utils.py:13` is exactly this expression, importing the handler on a canonical checkout fails at that line with the same `AttributeError`; the import chain is `email_handler.py:92` (`from app.email.spam import get_spam_score`) → `app/email/spam.py:11` (`from app.spamassassin_utils import SpamAssassin`) → `app/spamassassin_utils.py:13`. This inference is corroborated by the isolated reproduction above, which raises the identical error from the identical expression.

> **Image-baseline caveat [OBSERVED].** In *this* image the in-container working tree of `/app/app/spamassassin_utils.py` carries a **baseline patch** (present before any action of this investigation) that changes line 8 to the stdlib `import re` (which *does* expose `DOTALL`), masking the failure inside this particular image. This is confirmed read-only against the container's own git:
>
> ```bash
> docker exec "$CID" bash -lc 'cd /app && git diff -- app/spamassassin_utils.py'
> ```
>
> ```
> diff --git a/app/spamassassin_utils.py b/app/spamassassin_utils.py
> index f1e2d54c..d8a2d7ef 100644
> --- a/app/spamassassin_utils.py
> +++ b/app/spamassassin_utils.py
> @@ -5,7 +5,7 @@ import logging
>  import socket
>  from io import BytesIO
>
> -import re2 as re
> +import re
>  import select
>
>  from app.log import LOG
> ```
>
> The image ships several such baseline-modified files (six `local_data/*` example keys, `static/package-lock.json`, and this one); none was modified by this investigation. The **canonical** source — and the host deliverable checkout at this commit — use `import re2 as re` (shown by `git show` above), so a canonical checkout fails exactly as inferred. `pyre2` is the correct `poetry.lock`-consistent remedy regardless (`pyproject.toml` requires `pyre2 = "^0.3.6"`).

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

`re2` is now the compiled `pyre2` extension exposing `DOTALL`/`IGNORECASE`, and `google-re2` is absent. `import email_handler` then succeeds — this check requires `.env` to exist (`app/config.py:79` reads `os.environ["URL"]`), so it is run **after** §2.6 creates `.env`:

```bash
docker exec "$CID" bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/python -c "import email_handler; print(\"import email_handler OK\")"'
```

Output **[OBSERVED]** (complete; the app-config banner precedes the final line):

```
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/mslxmrgguhhxrfjkxmpn
Upload files to local dir
>>> init logging <<<
2026-07-13 23:31:17,569 - SL - DEBUG - 1270 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
import email_handler OK
```

#### 2.3.2 Install `swaks` (SMTP injection client)

The pristine image has **no** `swaks`, yet `swaks` is the sole canonical injector used from §3 onward. Install it once:

```bash
docker exec "$CID" bash -lc 'apt-get update && apt-get install -y swaks'
```

Verification command and output **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'command -v swaks && swaks --version | head -1'
```

```
/usr/bin/swaks
swaks version 20201014.0
```

`swaks` is a pure client (it makes no product change) and, like the `pyre2` swap, is a **setup** correction to the environment — no product file is modified.

### 2.4 Services: Postgres, Redis, MailHog

The image ships a Postgres 15 cluster (`main`) configured for the default port `5432` with only the roles `postgres`/`test` and databases `postgres`/`template0`/`template1`/`test`; the app role `myuser` and database `simplelogin` do **not** exist yet. Because `DB_URI` targets `myuser:…@localhost:15432/simplelogin` (§2.6), the cluster is moved to `15432`, started, and provisioned with the app role and database; Redis is started; and the MailHog sink (§2.5) is launched.

Provisioning and startup commands **[OBSERVED]** (the password is derived from `example.env` into `$PGPASS` and never printed):

```bash
# Postgres: move cluster 'main' from its default 5432 to 15432, start it,
# then create the app role and database (the pristine image has neither).
docker exec "$CID" bash -lc 'pg_conftool 15 main set port 15432 && pg_ctlcluster 15 main start'
docker exec "$CID" bash -lc '
  PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"
  su postgres -c "psql -p 15432 -v ON_ERROR_STOP=1 -c \"CREATE ROLE myuser SUPERUSER LOGIN PASSWORD '"'"'${PGPASS}'"'"';\""
  su postgres -c "psql -p 15432 -v ON_ERROR_STOP=1 -c \"CREATE DATABASE simplelogin OWNER myuser;\""
'
# Redis
docker exec "$CID" bash -lc 'redis-server --daemonize yes --port 6379'
```

Output of the Postgres role/database provisioning **[OBSERVED]** — the two `CREATE` results prove the role and database did **not** pre-exist (they would error otherwise):

```
CREATE ROLE
CREATE DATABASE
```

Resulting cluster state **[OBSERVED]** — now online on `15432`:

```bash
docker exec "$CID" bash -lc 'pg_lsclusters'
```

```
Ver Cluster Port  Status Owner    Data directory              Log file
15  main    15432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

Resulting service state:

- **Postgres 15.13** on port `15432`, role `myuser` (SUPERUSER), database `simplelogin` owned by `myuser`. The `pg_trgm` extension is **not** pre-created — it is created by the migration on a clean database (pre-creating it makes migration `2021_082012_424808e1fe49` roll back on `DUPLICATE_OBJECT`).
- **Redis 7.x** on `6379` (`redis-server --daemonize yes --port 6379`).
- **MailHog** SMTP sink on `127.0.0.1:1025`, capture API/UI on `127.0.0.1:1080` (§2.5).

Readiness commands and output **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "select '"'"'pg_ok'"'"', count(*) from pg_stat_activity;" | head -1'
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
sink banner: 220 mailhog.example ESMTP MailHog
```

The sink banner `220 mailhog.example ESMTP MailHog` confirms the downstream MTA is **MailHog** (the AAP/`CONTRIBUTING.md`-specified sink), not a bypassing stand-in.

### 2.5 SMTP sink: MailHog (canonical), install and capture

The Agent Action Plan and `CONTRIBUTING.md:184-221` specify **MailHog / mailcatcher** as the downstream sink for local email observation. MailHog is not preinstalled in the image, so the single upstream release binary is fetched (the container has internet) and run as the sink; it performs **zero transformation** and exposes every accepted message verbatim through its JSON API, so the delivered `From` and `Message-ID` can be read byte-for-byte.

Install and start **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc '
  wget -q -O /usr/local/bin/MailHog \
    https://github.com/mailhog/MailHog/releases/download/v1.0.1/MailHog_linux_amd64
  chmod +x /usr/local/bin/MailHog
'
docker exec -d "$CID" bash -lc '
  export MH_SMTP_BIND_ADDR=127.0.0.1:1025 MH_API_BIND_ADDR=127.0.0.1:1080 \
         MH_UI_BIND_ADDR=127.0.0.1:1080 MH_STORAGE=memory
  nohup /usr/local/bin/MailHog >/tmp/mailhog.log 2>&1 &
'
```

MailHog version and SMTP banner **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc '/usr/local/bin/MailHog --version 2>&1 | head -1'
docker exec "$CID" bash -lc 'python3 -c "import socket;s=socket.create_connection((\"127.0.0.1\",1025),timeout=3);print(s.recv(100).decode().strip());s.close()"'
```

```
MailHog version: 1.0.1
220 mailhog.example ESMTP MailHog
```

Delivered messages are read back with the capture API (`GET http://127.0.0.1:1080/api/v2/messages`). After the three byte-identical `hey@google.com → e1@sl.local` forwards (§4–§7) and the three Q3 new-sender forwards (§6.3), the API reports **six** delivered messages; the per-message envelope-from and key headers **[OBSERVED]**:

```
msg#1 | Message-ID=<blitzy-fixed-probe-2cd6ee777f8c@test.local> | X-SimpleLogin-EmailLog-ID=2 | From="hey at google.com" <hey_at_google_com_lvpak@sl.local> | envfrom=sl.lmycyibsfqqdemzygmytkns5.4qnl6zcsmp7fm@sl.local
msg#2 | Message-ID=<blitzy-fixed-probe-2cd6ee777f8c@test.local> | X-SimpleLogin-EmailLog-ID=3 | From="hey at google.com" <hey_at_google_com_lvpak@sl.local> | envfrom=sl.lmycyibtfqqdemzygmytkns5.x4e32pc3tvjue@sl.local
msg#3 | Message-ID=<blitzy-fixed-probe-2cd6ee777f8c@test.local> | X-SimpleLogin-EmailLog-ID=4 | From="hey at google.com" <hey_at_google_com_lvpak@sl.local> | envfrom=sl.lmycyibufqqdemzygmytkns5.upd6rptyysn5s@sl.local
```

Messages `#1–#3` are the three byte-identical runs (§4–§7); `#4–#6` are the Q3 new-sender sub-experiment (§6.3). The non-existent-alias failure (§4.2) delivered **nothing** — the MailHog count stayed at 3 (proof of no downstream egress, §4.2).

### 2.6 Configuration: `.env` deltas and resolution

`.env` is derived from `example.env` with exactly four deltas so a real forward is emitted to the local sink. Create it with `cp` plus four `sed` edits:

```bash
docker exec "$CID" bash -lc 'cd /app && cp example.env .env && \
  sed -i "s|^NOT_SEND_EMAIL=true|# NOT_SEND_EMAIL=true  # commented out: presence-based; real forward required|" .env && \
  sed -i "s|^# POSTFIX_SERVER=my-postfix.com|POSTFIX_SERVER=localhost|" .env && \
  sed -i "s|@localhost:5432/simplelogin|@localhost:15432/simplelogin|" .env && \
  sed -i "s|^# POSTFIX_PORT=1025|POSTFIX_PORT=1025|" .env'
```

The `DB_URI` delta changes only the port (`5432`→`15432`); the credential in the URI is left untouched by that substitution and is never written out in this report. The result is exactly four deltas, confirmed with `diff` **[OBSERVED]** (the `DB_URI` password is redacted here):

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
< DB_URI=postgresql://myuser:<redacted>@localhost:5432/simplelogin
---
> DB_URI=postgresql://myuser:<redacted>@localhost:15432/simplelogin
154c154
< # POSTFIX_PORT=1025
---
> POSTFIX_PORT=1025
```

`NOT_SEND_EMAIL` and `ENABLE_SPAM_ASSASSIN` are **presence-based** flags (`app/config.py:91`, `:450`): commenting `NOT_SEND_EMAIL` out makes it resolve to `False`, enabling a genuine send. Resolved configuration **[OBSERVED]** (`DB_URI` redacted at print time):

```bash
docker exec "$CID" bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/python -c "
from app import config
import re as _re
print(\"NOT_SEND_EMAIL =\", config.NOT_SEND_EMAIL)
print(\"POSTFIX_SERVER =\", config.POSTFIX_SERVER)
print(\"POSTFIX_PORT   =\", config.POSTFIX_PORT)
print(\"EMAIL_DOMAIN   =\", config.EMAIL_DOMAIN)
print(\"DB_URI         =\", _re.sub(r\"//([^:]+):[^@]+@\", r\"//\1:<redacted>@\", config.DB_URI))
"'
```

```
NOT_SEND_EMAIL = False
POSTFIX_SERVER = localhost
POSTFIX_PORT   = 1025
EMAIL_DOMAIN   = sl.local
DB_URI         = postgresql://myuser:<redacted>@localhost:15432/simplelogin
```

### 2.7 Schema migration and fixture seed

Migration `CONFIG=/app/.env alembic upgrade head` exited `0` and produced 265 output lines: the 10-line application/alembic banner (shown below verbatim) followed by exactly **255 sequential** `Running upgrade` transition lines. The banner plus the first two and last four transitions are shown here as a labeled **[EXCERPT]**; the **complete** 255-line transition sequence is reproduced verbatim in **Appendix A (§12)**, and migration completion is additionally proven by a hard post-migration state check (head revision, table count, `pg_trgm`).

Banner and boundary transitions — **[OBSERVED] [EXCERPT]** (full sequence: Appendix A):

```
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/zwishxfuwqjzhzpzlqhi
Upload files to local dir
>>> init logging <<<
2026-07-13 23:15:14,026 - SL - DEBUG - 567 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

(The first two lines `-> 5e549314e1e2` and `5e549314e1e2 -> 3cd10cfce8c3` and the final four transitions are the boundaries of the 255-line sequence shown in full in Appendix A.)

Post-migration state verification **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/alembic current 2>&1 | tail -1'
docker exec "$CID" bash -lc 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
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
WARNING: Use a temp directory for GNUPGHOME /tmp/wmzyyslmyazfhnvhgvyi
Upload files to local dir
>>> init logging <<<
2026-07-13 23:15:30,436 - SL - DEBUG - 603 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 23:15:30,900 - SL - WARNING - 603 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-13 23:15:30,900 - SL - DEBUG - 603 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-13 23:15:31,224 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:31,229 - SL - DEBUG - 603 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 23:15:31,251 - SL - DEBUG - 603 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_list<random>@sl.local
2026-07-13 23:15:31,262 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:31,314 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:31,322 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:31,340 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:31,351 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:31,363 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:31,371 - SL - DEBUG - 603 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-<random>
2026-07-13 23:15:31,378 - SL - DEBUG - 603 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-<random>
2026-07-13 23:15:31,655 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:31,657 - SL - DEBUG - 603 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 23:15:31,679 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:32,024 - SL - INFO - 603 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 23:15:32,034 - SL - INFO - 603 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

**Seeded fixtures used by the reproduction [OBSERVED]:** user `john@wick.com` (id 1, `include_sender_in_reverse_alias = t`); alias `e1@sl.local` = **alias id 5** (the success target). A pre-seeded baseline `Contact` (id 1) is `hey@google.com` on a *different* alias (alias_id 2) with a fixed seed reply-email `rep@sl.local`, so the first `hey@google.com → e1@sl.local` forward creates a **new** contact (relevant to Q4).

### 2.8 Starting the canonical entry point

Command (inside container):

```bash
docker exec -d "$CID" bash -lc 'cd /app && nohup env CONFIG=/app/.env /app/venv/bin/python email_handler.py >/tmp/handler.log 2>&1 &'
```

Startup lines **[OBSERVED]**:

```
2026-07-13 23:16:02,078 - SL - INFO - 683 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 23:16:02,079 - SL - DEBUG - 683 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
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

The payload is `314` bytes, CRLF-terminated, with exactly one `Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>`. All three success runs inject this **same file** (inside the container, per §2.1) via:

```bash
docker exec "$CID" bash -lc 'swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 --data - < /tmp/payload.eml'
```

**Byte-identity of what the handler actually received [OBSERVED].** The handler logs the parsed inbound envelope on one line (`==>> Handle ...`, `email_handler.py:1980`). Hashing that exact line for each of the three runs yields the identical digest, proving the handler processed identical input every time:

```bash
for u in 641bce01 8f09c3d9 ca681afd; do
  line=$(grep -m1 "${u}.*==>> Handle" /tmp/handler.log | sed 's/^.*==>> Handle/==>> Handle/')
  printf "%s inbound-line-sha256: %s\n" "$u" "$(printf '%s' "$line" | sha256sum | cut -d' ' -f1)"
done
```

```
641bce01 inbound-line-sha256: 15018c27637ceb5bdc64eed42ea7e8d53889a8e97b7959bf9fd59466f9d6cc60
8f09c3d9 inbound-line-sha256: 15018c27637ceb5bdc64eed42ea7e8d53889a8e97b7959bf9fd59466f9d6cc60
ca681afd inbound-line-sha256: 15018c27637ceb5bdc64eed42ea7e8d53889a8e97b7959bf9fd59466f9d6cc60
```

Identical digest across all three runs ⇒ **byte-identical input**, satisfying the "replay the SAME unchanged input" discipline.

---

## 4. Q1 — Log message text (success vs. non-existent alias)

### 4.1 Success — complete handler stdout (run #1, first send)

Command (inside container) that injected run #1:

```bash
docker exec "$CID" bash -lc 'swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 --data - < /tmp/payload.eml'
```

Complete SMTP transcript returned to the client **[OBSERVED]** — terminal status is `250 Message accepted for delivery`:

```
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 8f131273a9f0 Python SMTP 1.4.2
 -> EHLO 8f131273a9f0
<-  250-8f131273a9f0
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

Complete handler stdout for run #1 **[OBSERVED]** (log-tracing id `641bce01-2463-4d39-a349-a4cfb51c66ec`):

```
2026-07-13 23:16:25,197 - SL - DEBUG - 683 - "/app/app/log.py:24" - set_message_id() -  - set message_id 641bce01-2463-4d39-a349-a4cfb51c66ec
2026-07-13 23:16:25,197 - SL - DEBUG - 683 - "/app/email_handler.py:2342" - _handle() - 641bce01-2463-4d39-a349-a4cfb51c66ec - ====>=====>====>====>====>====>====>====>
2026-07-13 23:16:25,197 - SL - INFO - 683 - "/app/email_handler.py:2343" - _handle() - 641bce01-2463-4d39-a349-a4cfb51c66ec - New message, mail from hey@google.com, rctp tos ['e1@sl.local']
2026-07-13 23:16:25,199 - SL - DEBUG - 683 - "/app/email_handler.py:1963" - handle() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Cannot parse Postfix queue ID from None None
2026-07-13 23:16:25,361 - SL - DEBUG - 683 - "/app/email_handler.py:1980" - handle() - 641bce01-2463-4d39-a349-a4cfb51c66ec - ==>> Handle mail_from:hey@google.com, rcpt_tos:['e1@sl.local'], header_from:hey@google.com, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<blitzy-fixed-probe-2cd6ee777f8c@test.local>, client_ip:None, headers:[('Date', 'Mon, 13 Jul 2026 12:00:00 +0000'), ('To', 'e1@sl.local'), ('From', 'hey@google.com'), ('Subject', 'Blitzy fixed forward probe'), ('Message-ID', '<blitzy-fixed-probe-2cd6ee777f8c@test.local>'), ('Content-Type', 'text/plain; charset=UTF-8'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 23:16:25,366 - SL - DEBUG - 683 - "/app/email_handler.py:2202" - handle() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Forward phase hey@google.com(hey@google.com) -> e1@sl.local
2026-07-13 23:16:25,381 - SL - DEBUG - 683 - "/app/email_handler.py:580" - handle_forward() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Create or get contact for from_header:hey@google.com
2026-07-13 23:16:25,407 - SL - DEBUG - 683 - "/app/app/contact_utils.py:110" - create_contact() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Created contact <Contact 2 hey@google.com 5> for alias <Alias 5 e1@sl.local> with email hey@google.com invalid_email=False
2026-07-13 23:16:25,407 - SL - INFO - 683 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 641bce01-2463-4d39-a349-a4cfb51c66ec - DMARC check disabled
2026-07-13 23:16:25,415 - SL - DEBUG - 683 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Forward <Contact 2 hey@google.com 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-13 23:16:25,419 - SL - DEBUG - 683 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Create <EmailLog 2> for <Contact 2 hey@google.com 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-13 23:16:25,425 - SL - DEBUG - 683 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 641bce01-2463-4d39-a349-a4cfb51c66ec - From header, new:"hey at google.com" <hey_at_google_com_lvpak@sl.local>, old:hey@google.com
2026-07-13 23:16:25,425 - SL - DEBUG - 683 - "/app/email_handler.py:316" - replace_header_when_forward() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Delete Cc header, old value None
2026-07-13 23:16:25,425 - SL - DEBUG - 683 - "/app/email_handler.py:313" - replace_header_when_forward() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Replace To header, old: e1@sl.local, new: e1@sl.local
2026-07-13 23:16:25,425 - SL - INFO - 683 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Email has no unsubscribe header
2026-07-13 23:16:25,425 - SL - DEBUG - 683 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Forward mail from hey@google.com to john@wick.com, mail_options:[], rcpt_options:[]
2026-07-13 23:16:25,426 - SL - DEBUG - 683 - "/app/app/mail_sender.py:156" - _send_to_smtp() - 641bce01-2463-4d39-a349-a4cfb51c66ec - getting a smtp connection takes seconds 0.0006422996520996094
2026-07-13 23:16:25,426 - SL - DEBUG - 683 - "/app/app/mail_sender.py:163" - _send_to_smtp() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Sendmail mail_from:sl.lmycyibsfqqdemzygmytkns5.4qnl6zcsmp7fm@sl.local, rcpt_to:john@wick.com, header_from:"hey at google.com" <hey_at_google_com_lvpak@sl.local>, header_to:e1@sl.local, header_cc:None
2026-07-13 23:16:25,428 - SL - INFO - 683 - "/app/email_handler.py:2367" - _handle() - 641bce01-2463-4d39-a349-a4cfb51c66ec - Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.23096871376037598 seconds with return code '250 Message accepted for delivery'<<===
```

**Success log lines that answer Q1 [OBSERVED]:**

| Log text (verbatim) | Level | Emitted at (observed `file:line`) |
|---------------------|-------|-----------------------------------|
| `Forward phase hey@google.com(hey@google.com) -> e1@sl.local` | DEBUG | `email_handler.py:2202` |
| `Created contact <Contact 2 hey@google.com 5> for alias <Alias 5 e1@sl.local> with email hey@google.com invalid_email=False` | DEBUG | `app/contact_utils.py:110` |
| `Create <EmailLog 2> for <Contact 2 hey@google.com 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>` | DEBUG | `email_handler.py:740` |
| `From header, new:"hey at google.com" <hey_at_google_com_lvpak@sl.local>, old:hey@google.com` | DEBUG | `email_handler.py:867` |
| `Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.23096871376037598 seconds with return code '250 Message accepted for delivery'<<===` | INFO | `email_handler.py:2367` |

The terminating status string `250 Message accepted for delivery` is the constant `E200` (`app/email/status.py:2`) **[INFERRED from source; the string itself is OBSERVED in the summary line above]**. The `getting a smtp connection takes seconds` line (`app/mail_sender.py:156`) is emitted **on the real send path** after the SMTP connection is obtained — it appears because `NOT_SEND_EMAIL=False` routes execution through `_send_to_smtp` (`app/mail_sender.py:144`) rather than the `NOT_SEND_EMAIL` short-circuit (`app/mail_sender.py:130-136`).


### 4.2 Non-existent alias — complete handler stdout and no-egress proof

**Pre-send absence check [OBSERVED]** — the target alias does not exist before the send:

```bash
docker exec "$CID" bash -lc 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "select count(*) from alias where email='"'"'doesnotexist@sl.local'"'"';"'
```

```
0
```

Command (inside container) — same fixed payload, only the RCPT changes to a non-existent alias:

```bash
docker exec "$CID" bash -lc 'swaks --to doesnotexist@sl.local --from hey@google.com --server 127.0.0.1:20381 --data - < /tmp/payload.eml'
```

Complete SMTP transcript **[OBSERVED]** — terminal status is `550 SL E515 Email not exist` (note `<**`, an SMTP error reply):

```
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 8f131273a9f0 Python SMTP 1.4.2
 -> EHLO 8f131273a9f0
<-  250-8f131273a9f0
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

Complete handler stdout for the failure **[OBSERVED]** (log-tracing id `dce933c3-5130-4452-8285-bc41de774962`):

```
2026-07-13 23:16:29,621 - SL - DEBUG - 683 - "/app/app/log.py:24" - set_message_id() - ca681afd-7a5d-4b4e-b8e7-5adb858813e5 - set message_id dce933c3-5130-4452-8285-bc41de774962
2026-07-13 23:16:29,622 - SL - DEBUG - 683 - "/app/email_handler.py:2342" - _handle() - dce933c3-5130-4452-8285-bc41de774962 - ====>=====>====>====>====>====>====>====>
2026-07-13 23:16:29,622 - SL - INFO - 683 - "/app/email_handler.py:2343" - _handle() - dce933c3-5130-4452-8285-bc41de774962 - New message, mail from hey@google.com, rctp tos ['doesnotexist@sl.local']
2026-07-13 23:16:29,622 - SL - DEBUG - 683 - "/app/email_handler.py:1963" - handle() - dce933c3-5130-4452-8285-bc41de774962 - Cannot parse Postfix queue ID from None None
2026-07-13 23:16:29,624 - SL - DEBUG - 683 - "/app/email_handler.py:1980" - handle() - dce933c3-5130-4452-8285-bc41de774962 - ==>> Handle mail_from:hey@google.com, rcpt_tos:['doesnotexist@sl.local'], header_from:hey@google.com, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<blitzy-fixed-probe-2cd6ee777f8c@test.local>, client_ip:None, headers:[('Date', 'Mon, 13 Jul 2026 12:00:00 +0000'), ('To', 'e1@sl.local'), ('From', 'hey@google.com'), ('Subject', 'Blitzy fixed forward probe'), ('Message-ID', '<blitzy-fixed-probe-2cd6ee777f8c@test.local>'), ('Content-Type', 'text/plain; charset=UTF-8'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 23:16:29,627 - SL - DEBUG - 683 - "/app/email_handler.py:2202" - handle() - dce933c3-5130-4452-8285-bc41de774962 - Forward phase hey@google.com(hey@google.com) -> doesnotexist@sl.local
2026-07-13 23:16:29,633 - SL - DEBUG - 683 - "/app/email_handler.py:545" - handle_forward() - dce933c3-5130-4452-8285-bc41de774962 - alias doesnotexist@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 23:16:29,639 - SL - INFO - 683 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - dce933c3-5130-4452-8285-bc41de774962 - Cannot auto-create custom domain alias for doesnotexist@sl.local because there's no custom domain for sl.local
2026-07-13 23:16:29,639 - SL - INFO - 683 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - dce933c3-5130-4452-8285-bc41de774962 - Cannot auto-create doesnotexist@sl.local since it has no directory separator
2026-07-13 23:16:29,640 - SL - DEBUG - 683 - "/app/email_handler.py:551" - handle_forward() - dce933c3-5130-4452-8285-bc41de774962 - alias doesnotexist@sl.local cannot be created on-the-fly, return 550
2026-07-13 23:16:29,641 - SL - INFO - 683 - "/app/email_handler.py:2367" - _handle() - dce933c3-5130-4452-8285-bc41de774962 - Finish mail_from hey@google.com, rcpt_tos ['doesnotexist@sl.local'], takes 0.019239425659179688 seconds with return code '550 SL E515 Email not exist'<<===
```

**Non-existent-alias log lines that answer Q1 [OBSERVED]:**

| Log text (verbatim) | Level | Emitted at (observed `file:line`) |
|---------------------|-------|-----------------------------------|
| `alias doesnotexist@sl.local not exist. Try to see if it can be created on the fly` | DEBUG | `email_handler.py:545` |
| `Cannot auto-create custom domain alias for doesnotexist@sl.local because there's no custom domain for sl.local` | INFO | `app/alias_utils.py:104` |
| `Cannot auto-create doesnotexist@sl.local since it has no directory separator` | INFO | `app/alias_utils.py:165` |
| `alias doesnotexist@sl.local cannot be created on-the-fly, return 550` | DEBUG | `email_handler.py:551` |
| `Finish mail_from hey@google.com, rcpt_tos ['doesnotexist@sl.local'], takes 0.019239425659179688 seconds with return code '550 SL E515 Email not exist'<<===` | INFO | `email_handler.py:2367` |

The terminating status string `550 SL E515 Email not exist` is the constant `E515` (`app/email/status.py:51`) **[INFERRED from source; the string itself is OBSERVED above]**. The `return 550` branch returns `[(False, status.E515)]` (`email_handler.py:554-555`) **[INFERRED]**.

**No-egress / no-side-effects proof [OBSERVED].** After the failure, all four forward-related tables are unchanged and MailHog stored nothing new. Command and output (this failure ran after the three success runs, so the counts reflect that state):

```bash
docker exec "$CID" bash -lc 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select '"'"'contact'"'"', count(*) from contact
union all select '"'"'email_log'"'"', count(*) from email_log
union all select '"'"'user_audit_log'"'"', count(*) from user_audit_log
union all select '"'"'message_id_matching'"'"', count(*) from message_id_matching;"'
docker exec "$CID" bash -lc 'wget -qO- http://127.0.0.1:1080/api/v2/messages | /app/venv/bin/python -c "import json,sys; print(\"mailhog_delivered:\", json.load(sys.stdin)[\"total\"])"'
```

```
contact|2
email_log|4
user_audit_log|1
message_id_matching|0
mailhog_delivered: 3
```

The counts `(contact 2, email_log 4, user_audit_log 1, message_id_matching 0)` are **identical to the post-run-#3 state** (§7.1), and MailHog still holds only the **3** successful forwards — the failed send created **no database rows and emitted no downstream message**.

### 4.3 Correlation-id (log-tracing id) semantics — with live proof

`set_message_id()` logs **before** it assigns the module global: `LOG.d("set message_id %s", message_id)` at `app/log.py:24` runs before `_MESSAGE_ID = message_id` at `app/log.py:25`. The formatter's filter runs at emit time (`EmailHandlerFilter.filter`, `app/log.py:31-34`) and reads `_MESSAGE_ID` via `get_message_id()` (`app/log.py:36-37`), so the **setter line itself carries the *previous* id** (blank on the very first message), and the **new** id appears from the **following** line onward. This was confirmed live four times **[OBSERVED]**:

| Event | Setter line prefix (prior id) | New id set on next line |
|-------|-------------------------------|--------------------------|
| run #1 (first message) | *(blank)* | `641bce01-2463-4d39-a349-a4cfb51c66ec` |
| run #2 | `641bce01-2463-4d39-a349-a4cfb51c66ec` | `8f09c3d9-b13a-408a-afcd-7497c9955e7b` |
| run #3 | `8f09c3d9-b13a-408a-afcd-7497c9955e7b` | `ca681afd-7a5d-4b4e-b8e7-5adb858813e5` |
| failure | `ca681afd-7a5d-4b4e-b8e7-5adb858813e5` | `dce933c3-5130-4452-8285-bc41de774962` |

(See the first line of each handler block in §4.1 and §4.2.) This log-tracing id is `str(uuid.uuid4())` created in `_handle()` (`email_handler.py:2339`), passed to `set_message_id()` (`email_handler.py:2340`); when `handle()` runs under Postfix it is instead the queue id (`email_handler.py:1959`), which here logs `Cannot parse Postfix queue ID from None None` (`email_handler.py:1963`) because `swaks` supplies no queue id. This id is **not** an email `Message-ID` (Q2).

---

## 5. Q2 — SL Message-ID vs. original Message-ID

The question conflates what the running code treats as **three distinct identifiers**. The investigation observed each and disambiguates them.

### 5.1 The three identifiers

| # | Identifier | Where it comes from | Observed on this forward? |
|---|-----------|---------------------|----------------------------|
| 1 | **Original `Message-ID` header** — `<blitzy-fixed-probe-2cd6ee777f8c@test.local>` | Set by the sender; **preserved** by the forward path (`email_handler.py:799` comment `# do not delete original message id`; the header is kept via `headers_to_keep`, `email_handler.py:793`). | **[OBSERVED]** preserved unchanged in the delivered message. |
| 2 | **Log-tracing id** — a per-message `uuid4` (run #1 value `641bce01-2463-4d39-a349-a4cfb51c66ec`) | `str(uuid.uuid4())` in `_handle()` (`email_handler.py:2339`), injected into every log line as `%(message_id)s` (`app/log.py:14`). | **[OBSERVED]** varies every message (§4.3). Not an email header. |
| 3 | **`sl_message_id`** — a `make_msgid(...)` value | Built in the **reply** path `replace_original_message_id()` (`email_handler.py:1311`), persisted in `message_id_matching` (`MessageIDMatching`, `app/models.py:3365`, column `sl_message_id` `app/models.py:3371`) and `email_log.sl_message_id` (`app/models.py:2116`). | **[OBSERVED] NOT created by the forward** — see §5.3. |

### 5.2 The delivered `Message-ID` equals the sent one (byte-for-byte)

MailHog captured the delivered message for each of the three identical runs. Reading the key headers back from the capture API, the `Message-ID` is identical to the sent one in every run **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc '/app/venv/bin/python - <<PY
import json, urllib.request
d=json.load(urllib.request.urlopen("http://127.0.0.1:1080/api/v2/messages"))
items=list(reversed(d["items"]))[:3]   # oldest first, the 3 identical runs
for i,it in enumerate(items,1):
    raw=it["Raw"]["Data"].replace("\r\n","\n")
    print("--- msg#%d ---" % i)
    for line in raw.split("\n"):
        if line.startswith(("Message-ID:","From:","X-SimpleLogin-EmailLog-ID:")):
            print(line)
        if line.strip()=="":break
PY'
```

```
--- msg#1 ---
Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
X-SimpleLogin-EmailLog-ID: 2
From: "hey at google.com" <hey_at_google_com_lvpak@sl.local>
--- msg#2 ---
Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
X-SimpleLogin-EmailLog-ID: 3
From: "hey at google.com" <hey_at_google_com_lvpak@sl.local>
--- msg#3 ---
Message-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>
X-SimpleLogin-EmailLog-ID: 4
From: "hey at google.com" <hey_at_google_com_lvpak@sl.local>
```

Sent `Message-ID` = `<blitzy-fixed-probe-2cd6ee777f8c@test.local>` (§3); delivered `Message-ID` = the same, for all three runs. **Difference = zero bytes.** Direct proof the forward path does **not** rewrite the `Message-ID` **[OBSERVED]**. (The forward path only rewrites threading headers `In-Reply-To`/`References` via `replace_sl_message_id_by_original_message_id()`, defined `email_handler.py:931`, called `email_handler.py:860` — it does **not** touch `Message-ID` **[INFERRED]**.)

The delivered messages also confirm there is **exactly one** `Message-ID` header (and exactly one `From`). Full delivered header block for run #1, read from MailHog **[OBSERVED]**:

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
From: "hey at google.com" <hey_at_google_com_lvpak@sl.local>
To: e1@sl.local

Fixed byte-identical body for run-to-run replay. Do not change.
```

There is **no** `DKIM-Signature` header in the delivered block (consistent with §8: the message is not DKIM-signed under this default configuration).

### 5.3 `sl_message_id` is a reply-phase artifact — not written by any forward

Labeled snapshots of `message_id_matching` **and** `email_log.sl_message_id` were taken before and after each transaction. `message_id_matching` stayed empty throughout, and `email_log.sl_message_id` is empty for every forwarded row **[OBSERVED]**:

```bash
# taken at each labeled checkpoint (PGPASS derived from example.env, never printed)
docker exec "$CID" bash -lc 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select '"'"'message_id_matching_count'"'"', count(*) from message_id_matching;"'
docker exec "$CID" bash -lc 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
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

The delivered `From` header, read from the MailHog capture API **[OBSERVED]**:

```
From: "hey at google.com" <hey_at_google_com_lvpak@sl.local>
```

This is identical across all three byte-identical runs (§5.2). The handler also logged the replacement inline **[OBSERVED]**:

```
2026-07-13 23:16:25,425 - SL - DEBUG - 683 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 641bce01-2463-4d39-a349-a4cfb51c66ec - From header, new:"hey at google.com" <hey_at_google_com_lvpak@sl.local>, old:hey@google.com
```

The new `From` value is produced by `contact.new_addr()` and assigned as `new_from_header` (`email_handler.py:865`), then logged at `email_handler.py:867` **[INFERRED for the code path; the value is OBSERVED]**.

### 6.2 Anatomy of the value

- **Display name** `"hey at google.com"` — the sender address `hey@google.com` with `@` → ` at `. This is `SenderFormatEnum.AT` (value `0`, the default; `app/models.py:2028-2029`), applied in `Contact.new_addr()` (`app/models.py:2008`) **[OBSERVED value; INFERRED mechanism]**.
- **Angle-addr** `<hey_at_google_com_lvpak@sl.local>` — the reverse-alias stored in `Contact.reply_email` (`app/models.py:1899`). Local part = `{sanitized_sender}_{random}`:
  - `hey_at_google_com` — the sender with non-alphanumerics collapsed to `_`.
  - `lvpak` — a random suffix (here 5 chars).
  - No `ra+`/`reply+` prefix — that legacy prefix is **commented out** in this version (`app/email_utils.py:1141`, `:1147`) **[INFERRED from source]**.
  - Domain `sl.local` = `EMAIL_DOMAIN` (§2.6).

### 6.3 Reverse-alias generation branch and length distribution

The seeded user `john@wick.com` has `include_sender_in_reverse_alias = True` (§2.7), so `generate_reply_email()` (`app/email_utils.py:1103`) takes the **sender-included** branch (`app/email_utils.py:1116-1117`), whose random suffix is `random_string(randint(5, 10))` (`app/email_utils.py:1138`, f-string `:1139-1143`) **[INFERRED mechanism]**. To observe the actual distribution, three additional **new senders** were forwarded to `e1@sl.local` (MailHog messages `#4–#6`), and the reverse-alias local-part lengths measured **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, website_email, reply_email, length(split_part(reply_email,'"'"'@'"'"',1)) as local_len
from contact where alias_id = 5 order by id;"'
```

```
2|hey@google.com|hey_at_google_com_lvpak@sl.local|23
3|alice@example.org|alice_at_example_org_xixiwgv@sl.local|28
4|bob@example.org|bob_at_example_org_itwchqfjl@sl.local|28
5|carol@example.org|carol_at_example_org_duqth@sl.local|26
```

Random suffix lengths (local-part after the final `_`): `lvpak`=5, `xixiwgv`=7, `itwchqfjl`=9, `duqth`=5 — all within `randint(5, 10)` **[OBSERVED]**. The suffix length (and therefore the whole local part) varies from one new sender to the next; only the *format* `{sanitized_sender}_{random}` is stable. (The pre-seeded baseline contact id 1 for `hey@google.com` on alias_id 2 carries the fixture reply-email `rep@sl.local`, which is a fake-data fixed value — not a `generate_reply_email` random suffix — so it is excluded by the `alias_id = 5` filter above.)

### 6.4 Cross-check: the transformed address is a valid reverse-alias (derived dynamically)

The reverse-alias is **not** hardcoded here; it is read back from the database for the `(sender, alias)` pair and then round-tripped through the product's own predicate. This makes the check reproducible on any fresh run **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc '
  export PGPASSWORD="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"
  RA=$(psql -h localhost -p 15432 -U myuser -d simplelogin -tAc \
       "select reply_email from contact where website_email='"'"'hey@google.com'"'"' and alias_id=5;")
  echo "$RA" > /tmp/derived_ra.txt
  echo "derived reverse-alias: $RA"
'
docker exec "$CID" bash -lc 'cd /app && RA=$(cat /tmp/derived_ra.txt) && CONFIG=/app/.env /app/venv/bin/python -c "
import sys
from app.email_utils import is_reverse_alias
ra = sys.argv[1].strip()
print(\"is_reverse_alias(reverse-alias):\", is_reverse_alias(ra))
print(\"is_reverse_alias(original sender):\", is_reverse_alias(\"hey@google.com\"))
" "$RA"'
```

```
derived reverse-alias: hey_at_google_com_lvpak@sl.local
is_reverse_alias(reverse-alias): True
is_reverse_alias(original sender): False
```

`is_reverse_alias(...)` (`app/email_utils.py:1156`) returns `True` for the delivered `From`'s angle-addr and `False` for the original sender address — confirming the delivered `From` is a valid reverse-alias for this `(sender, alias)` pair. The derived value `hey_at_google_com_lvpak@sl.local` matches the delivered `From` (§6.1) and `Contact.reply_email` for contact id 2 (§7.1).

---

## 7. Q4 — Database records created by a single forward

### 7.1 Before / after snapshots (full projections matching displayed columns)

Counts before any send, and row-level detail after each of the three identical runs, using SQL projections whose selected columns match the displayed columns exactly. (Every `psql` derives `$PGPASS` from `example.env` and passes `PGPASSWORD="$PGPASS"`; abbreviated below as `PGPASSWORD="$PGPASS"` for readability.)

**BEFORE any send [OBSERVED]:**

```bash
docker exec "$CID" bash -lc 'PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
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
docker exec "$CID" bash -lc 'PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, alias_id, website_email, reply_email, created_at from contact where id=2;"'
docker exec "$CID" bash -lc 'PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, user_id, action, message, created_at from user_audit_log where id=1;"'
docker exec "$CID" bash -lc 'PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, contact_id, user_id, mailbox_id, coalesce(message_id,'"'"'<empty>'"'"'), coalesce(sl_message_id,'"'"'<empty>'"'"'), created_at from email_log where id=2;"'
```

```
2|5|hey@google.com|hey_at_google_com_lvpak@sl.local|2026-07-13 23:16:25.397606
1|1|create_contact|Created contact 2 (hey@google.com)|2026-07-13 23:16:25.404026
2|2|1|1|<blitzy-fixed-probe-2cd6ee777f8c@test.local>|<empty>|2026-07-13 23:16:25.416958
```

**AFTER run #2 and run #3 (repeat sends, same sender) — only a new `EmailLog` each [OBSERVED]:**

```bash
docker exec "$CID" bash -lc 'PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select id, contact_id, user_id, mailbox_id, coalesce(message_id,'"'"'<empty>'"'"'), coalesce(sl_message_id,'"'"'<empty>'"'"'), created_at from email_log where id in (3,4) order by id;"'
docker exec "$CID" bash -lc 'PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "
select '"'"'contact'"'"', count(*) from contact union all select '"'"'email_log'"'"', count(*) from email_log union all select '"'"'user_audit_log'"'"', count(*) from user_audit_log union all select '"'"'message_id_matching'"'"', count(*) from message_id_matching;"'
```

```
3|2|1|1|<blitzy-fixed-probe-2cd6ee777f8c@test.local>|<empty>|2026-07-13 23:16:26.800347
4|2|1|1|<blitzy-fixed-probe-2cd6ee777f8c@test.local>|<empty>|2026-07-13 23:16:28.249848
contact|2
email_log|4
user_audit_log|1
message_id_matching|0
```

### 7.2 Records created by one forward — summary

| Send | Records created | IDs | `created_at` (observed) |
|------|-----------------|-----|--------------------------|
| **Run #1** (new sender) | `Contact`, `UserAuditLog`, `EmailLog` | 2, 1, 2 | `23:16:25.397606`, `23:16:25.404026`, `23:16:25.416958` |
| **Run #2** (repeat) | `EmailLog` only | 3 | `23:16:26.800347` |
| **Run #3** (repeat) | `EmailLog` only | 4 | `23:16:28.249848` |

Creation order within run #1 (by timestamp) **[OBSERVED]**: `Contact` (`.397606`) → `UserAuditLog` (`.404026`) → `EmailLog` (`.416958`). This matches the code order **[INFERRED]**: `create_contact()` creates the `Contact` (`app/contact_utils.py:91-102`) and emits the `UserAuditLog` (`app/contact_utils.py:104-109`), after which `forward_email_to_mailbox()` creates the `EmailLog` (`email_handler.py:732`, logged `email_handler.py:740`).

### 7.3 Why a repeat send reuses the `Contact` — app lookup vs. DB constraint (distinct)

Two independent mechanisms enforce one `Contact` per (alias, sender); they are **not** the same line and the report cites both:

- **Application-level reuse lookup [INFERRED]:** `create_contact()` first calls `Contact.get_by(alias_id=alias.id, website_email=email)` (`app/contact_utils.py:85`); on a hit it returns the existing contact instead of inserting — this is why runs #2/#3 add no `Contact` row.
- **Database-level uniqueness constraint [INFERRED]:** the `Contact` model declares `sa.UniqueConstraint("alias_id", "website_email", name="uq_contact")` (`app/models.py:1874-1876`, literal at `:1875`), created by migration `2020_031711_0809266d08ca_.py:45` (`op.create_unique_constraint("uq_contact", "contact", ["alias_id", "website_email"])`). This is the schema-enforced guarantee, independent of the app lookup.

### 7.4 ID and timestamp provenance

Every row's `id` is the autoincrement PK and `created_at` is `arrow.utcnow()`, both from `ModelMixin` (`app/models.py:62`; `id` `:63`; `created_at` `:64`) **[INFERRED mechanism; the concrete ids/timestamps above are OBSERVED]**. Consequently the `id` values and `created_at` timestamps are **run-to-run variable** by construction (§8).


---

## 8. Run-to-run stability (directly addressing "inconsistent behavior")

The **same 314-byte payload** (`sha256=c1dd2e00b7fc4009815f7b4a13b61151c367744ec480c5cfa2e599daf1f84471`) was replayed three times; the handler received byte-identical input each time (`inbound-line-sha256=15018c27637ceb5bdc64eed42ea7e8d53889a8e97b7959bf9fd59466f9d6cc60`, identical across all three runs — §3). Observed distribution across the three runs:

| Field | Run #1 | Run #2 | Run #3 | Classification |
|-------|--------|--------|--------|----------------|
| SMTP terminal status | `250 Message accepted for delivery` | same | same | **STABLE** [OBSERVED] |
| Success summary template (`email_handler.py:2367`) | present | present | present | **STABLE** [OBSERVED] |
| Delivered `Message-ID` | `<blitzy-fixed-probe-2cd6ee777f8c@test.local>` | same | same | **STABLE** [OBSERVED] |
| Delivered `From` format | `"hey at google.com" <hey_at_google_com_lvpak@sl.local>` | same | same | **STABLE** [OBSERVED] |
| `Contact.reply_email` (contact reused) | `hey_at_google_com_lvpak@sl.local` | same | same | **STABLE** [OBSERVED] |
| `EmailLog.id` | `2` | `3` | `4` | **VARIABLE** [OBSERVED] |
| `EmailLog.created_at` | `2026-07-13 23:16:25.416958` | `2026-07-13 23:16:26.800347` | `2026-07-13 23:16:28.249848` | **VARIABLE** [OBSERVED] |
| Log-tracing `uuid4` | `641bce01-2463-4d39-a349-a4cfb51c66ec` | `8f09c3d9-b13a-408a-afcd-7497c9955e7b` | `ca681afd-7a5d-4b4e-b8e7-5adb858813e5` | **VARIABLE** [OBSERVED] |
| Delivered envelope-from (VERP, `mail_sender.py:163`) | `sl.lmycyibsfqqdemzygmytkns5.4qnl6zcsmp7fm@sl.local` | `sl.lmycyibtfqqdemzygmytkns5.x4e32pc3tvjue@sl.local` | `sl.lmycyibufqqdemzygmytkns5.upd6rptyysn5s@sl.local` | **VARIABLE** [OBSERVED] |
| `X-SimpleLogin-EmailLog-ID` header | `2` | `3` | `4` | **VARIABLE** [OBSERVED] |
| MailHog-stored `.eml` sha256 | `cf9cfae512bd9bc74752d68e3089ffc286353e9e7d48c38edcaccb4a184fa173` | `21bba5c0c755668d69dc1f43a765c755a5ff3d29762118ee01a22d1b4bdc0b54` | `b1be637aad6a24c94736eebf9189f29686bf1144e3ce097e8806511377b888a7` | **VARIABLE** [OBSERVED] |

**Nuance worth highlighting [OBSERVED]:** even with byte-identical *input*, the full delivered `.eml` bytes differ every run, because the message embeds the fresh `EmailLog.id` in the `X-SimpleLogin-EmailLog-ID` header (`2` → `3` → `4` above). So "the forwarded email is not byte-identical run to run" is **true and expected** — while the two fields a recipient usually notices (`Message-ID`, `From` format) are stable. The message is **not** DKIM-signed under this default configuration: `DKIM_PRIVATE_KEY_PATH` is commented out (`example.env:72`), so `config.DKIM_PRIVATE_KEY` is `None` (`app/config.py:184`) and the unconditional `add_dkim_signature(msg, EMAIL_DOMAIN)` call (`email_handler.py:891`) is a no-op via the `if config.DKIM_PRIVATE_KEY:` guard (`app/email_utils.py:490`) — consistent with the §5.2 delivered header set, which contains no `DKIM-Signature`.

**Bounded conclusion / hypothesis (finding-aware):**

- **[OBSERVED]** On the **forward** path, with identical input, behavior is *format-stable* (templates, status codes, preserved `Message-ID`, `From`/reverse-alias format) and *value-variable* (`EmailLog.id`, timestamps, log-tracing uuid, VERP envelope-from, whole-message bytes — solely due to the `X-SimpleLogin-EmailLog-ID` increment; the message is not DKIM-signed by default).
- **[HYPOTHESIS]** A user perceiving "inconsistent behavior" is most plausibly seeing (i) these intentionally per-message-variable fields and/or (ii) **phase-dependent** `Message-ID` handling (preserved on forward vs. replaced on reply, §5).
- This report deliberately does **not** claim the reported production issue is "not a defect" or globally deterministic: the **reply** and **bounce** paths were not exercised here, so any statement about them would be unsupported. The scope of these conclusions is exactly the **forward** transactions observed above.

---

## 9. Official SimpleLogin context (product-level), with local-authoritative caveat

These official sources frame the reverse-alias and `From`-replacement concepts at the product level. **The running local code (cited throughout) remains the source of truth for exact values**; the public docs describe the hosted `.co` service and differ from the local runtime in the ways noted.

- **Reverse-alias overview** — SimpleLogin docs: [https://simplelogin.io/docs/getting-started/reverse-alias/](https://simplelogin.io/docs/getting-started/reverse-alias/). A reverse-alias is unique per (sender, alias) and by default composed of random characters — corroborating Q3's mechanism.
- **FAQ** — [https://simplelogin.io/faq/](https://simplelogin.io/faq/). Describes the reverse-alias as a special alias created per alias-and-contact that lets you send from your alias, i.e. the `From` a recipient sees is the alias/reverse-alias, not the real mailbox — corroborating the `From` replacement (`contact.new_addr()`).
- **Sending from an alias** — [https://simplelogin.io/docs/getting-started/send-email/](https://simplelogin.io/docs/getting-started/send-email/). Confirms replies target the reverse-alias (the reply-phase relevant to identifier #3 in §5).
- **Reverse-alias generation algorithm** — SimpleLogin blog: [https://simplelogin.io/blog/reverse-alias/](https://simplelogin.io/blog/reverse-alias/). Documents that reverse-aliases historically looked like `ra+<random>@simplelogin.co` and that a change later *included the sender address* in the reverse-alias. This directly explains the two format families in the code (the sender-included branch vs. the default branch in `generate_reply_email`).

**Local-runtime-authoritative caveat [OBSERVED vs. public docs]:** in this local build the domain is `sl.local` (not `simplelogin.co`); the `ra+` prefix is **commented out** (`app/email_utils.py:1141`, `:1147`), so the observed local reverse-alias is *prefix-less* `hey_at_google_com_lvpak@sl.local`; and the *sender-included* branch is active because the seeded user has `include_sender_in_reverse_alias = True`. Where public docs and local behavior differ, the **observed local values above govern**.

---

## 10. Coverage pass

| Question | Named item | Where answered | Label |
|----------|-----------|----------------|-------|
| Q1 | Success log text + level | §4.1 (INFO `Finish` summary ending `'250 Message accepted for delivery'<<===`, `email_handler.py:2367`; `E200` `app/email/status.py:2`) | [OBSERVED] |
| Q1 | Non-existent-alias log text + level | §4.2 (DEBUG `not exist` `:545`; DEBUG `cannot be created on-the-fly, return 550` `:551`; INFO `Finish` summary ending `'550 SL E515 Email not exist'<<===` `:2367`; `E515` `app/email/status.py:51`) | [OBSERVED] |
| Q2 | The "SL Message-ID" generated during forwarding | §5.2–5.4 (none applied on forward; original preserved, 0-byte diff) | [OBSERVED] |
| Q2 | How it differs from the original | §5.2 (identical — zero difference on forward); §5.1/5.3 (`sl_message_id` is a reply-phase artifact) | [OBSERVED]+[INFERRED] |
| Q3 | Exact transformed `From` header | §6.1 (`"hey at google.com" <hey_at_google_com_lvpak@sl.local>`) | [OBSERVED] |
| Q3 | Reverse-alias (reply-email) format | §6.2–6.4 (`{sanitized_sender}_{random 5–10}@sl.local`, prefix-less; `is_reverse_alias`=True) | [OBSERVED]+[INFERRED] |
| Q4 | Every record created by one forward | §7.1–7.2 (first: `Contact` 2 + `UserAuditLog` 1 + `EmailLog` 2; repeat: `EmailLog` only) | [OBSERVED] |
| Q4 | Actual record IDs and timestamps | §7.1 (Contact id 2, UserAuditLog id 1, EmailLog ids 2/3/4; timestamps from `2026-07-13 23:16:25.397606` through `2026-07-13 23:16:28.249848`) | [OBSERVED] |
| "inconsistent behavior" | Reproduce, don't stabilize | §3 (byte-identical replay), §8 (observed distribution, bounded hypothesis) | [OBSERVED]+[HYPOTHESIS] |

---

## 11. Cleanup & teardown (read-only guarantee restored)

The investigation was read-only against the SimpleLogin source and used only temporary, uniquely-named artifacts. All of them — the investigation container, its services (Postgres/Redis/MailHog, which live only inside the container), the host observation directory, and a gitignored `.env` created inside the container's `/app` — are removed. Teardown targets **only** validated, uniquely-named resources by their captured identifiers (mitigating path-traversal / wrong-target risk, CWE-22): the container is removed by its **exact captured id** after confirming it still carries the investigation label, `$REPO` is bound to the resolved host repository root, and each filesystem path is re-resolved with `realpath` and prefix-checked before removal.

**Pre-cleanup safety validation [OBSERVED]** — bind `$CID` and `$REPO`, then validate:

```bash
CID="$(cat /tmp/blitzy_inv_06d10137/container_id.txt)"
REPO="$(git -C /tmp/blitzy/app/blitzy-06d10137-d466-4563-b96c-72f05d800c84_94689c rev-parse --show-toplevel)"
echo "REPO=$REPO"
docker ps -a --filter "id=${CID}" --format '{{.ID}}  {{.Label "blitzy-investigation"}}  {{.Image}}  {{.Status}}'
```

```
REPO=/tmp/blitzy/app/blitzy-06d10137-d466-4563-b96c-72f05d800c84_94689c
8f131273a9f0  app_2cd6ee777f8c  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0  Up 41 minutes
```

The container id matches the captured value and carries the `blitzy-investigation=app_2cd6ee777f8c` label; `$REPO` resolves to the repository root. (There is no host `.env` to remove — `.env` was created only inside the container's `/app`; the host working tree contains no `.env`, and none is tracked: `.gitignore:4` lists `.env`.)

**Teardown [OBSERVED]** (each step with its exit status; the container removal takes the services down with it):

```bash
test -n "$CID" && docker rm -f "$CID"; echo "exit=$?"
case "$(realpath /tmp/blitzy_inv_06d10137)" in /tmp/blitzy_inv_06d10137) rm -rf /tmp/blitzy_inv_06d10137;; *) echo "refuse";; esac; echo "exit=$?"
```

```
8f131273a9f01305018f45e321a4c019dda61708780bd3822e42d05e3362ce89
exit=0
exit=0
```

**Post-cleanup verification [OBSERVED]:**

```bash
git -C "$REPO" status --porcelain
docker ps -a --filter "id=${CID}" --format '{{.ID}} {{.Status}}' | grep . || echo "(container absent — removed)"
docker ps -a --filter "label=blitzy-investigation=app_2cd6ee777f8c" --format '{{.ID}}' | grep . || echo "(no labelled investigation containers remain)"
[ -d /tmp/blitzy ] && echo "intact: /tmp/blitzy"; [ -d "$REPO/.git" ] && echo "intact: repo/.git"
```

```
 M blitzy/documentation/app_2cd6ee777f8c.md
(container absent — removed)
(no labelled investigation containers remain)
intact: /tmp/blitzy
intact: repo/.git
```

**Extended resource-absence checks [OBSERVED]** (host-level; the investigation's services ran *inside* the container and published no host ports, volumes, or networks):

```bash
docker volume ls  --filter "label=blitzy-investigation=app_2cd6ee777f8c" --format '{{.Name}}' | grep . || echo "(no investigation-labelled volumes)"
docker network ls --filter "label=blitzy-investigation=app_2cd6ee777f8c" --format '{{.Name}}' | grep . || echo "(no investigation-labelled networks)"
for p in 20381 1025 1080 15432 6379; do ss -ltn | grep -q ":$p " && echo "port $p: on host" || echo "port $p: not on host"; done
[ -e /tmp/blitzy_inv_06d10137 ] && echo "PRESENT" || echo "(/tmp/blitzy_inv_06d10137 absent)"
```

```
(no investigation-labelled volumes)
(no investigation-labelled networks)
port 20381: not on host
port 1025: not on host
port 1080: not on host
port 15432: not on host
port 6379: not on host
(/tmp/blitzy_inv_06d10137 absent)
```

No investigation-labelled Docker **volume** or **network** was ever created (the container ran with neither `-v` nor a custom `--network`), and none of the service **ports** (`20381` handler, `1025`/`1080` MailHog, `15432` Postgres, `6379` Redis) is bound on the host — confirming every service **process** lived inside the now-removed container. The only change in the working tree is this document (`blitzy/documentation/app_2cd6ee777f8c.md`); the investigation container and all label-matched containers are removed; and the shared workspace and repository git metadata are intact. The SimpleLogin product source is byte-for-byte unchanged.

> **Note on the commands above.** Every command block in this report is shown exactly as executed against the now-removed container (`$CID = 8f131273a9f0…`) and its temporary paths (`/tmp/payload.eml`, MailHog capture API). They are preserved as the reproducible record of what was run; the container and those paths no longer exist after this teardown. Re-running the ordered script in Appendix B reproduces the same environment.


---

## 12. Appendix A — Complete Alembic migration output (all 255 transitions)

This is the **complete, unedited** output of `CONFIG=/app/.env alembic upgrade head` referenced as an [EXCERPT] in §2.7 — the 10-line application/alembic banner followed by all **255** sequential `Running upgrade` transition lines (265 lines total). Command **[OBSERVED]**:

```bash
docker exec "$CID" bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/alembic upgrade head 2>&1'
```

Complete output **[OBSERVED]**:

```text
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/zwishxfuwqjzhzpzlqhi
Upload files to local dir
>>> init logging <<<
2026-07-13 23:15:14,026 - SL - DEBUG - 567 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 3cd10cfce8c3 -> 0256244cd7c8, empty message
INFO  [alembic.runtime.migration] Running upgrade 0256244cd7c8 -> 213fcca48483, empty message
INFO  [alembic.runtime.migration] Running upgrade 213fcca48483 -> f234688f5ebd, empty message
INFO  [alembic.runtime.migration] Running upgrade f234688f5ebd -> d03e433dc248, empty message
INFO  [alembic.runtime.migration] Running upgrade d03e433dc248 -> 2fe19381f386, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fe19381f386 -> b20ee72fd9a4, empty message
INFO  [alembic.runtime.migration] Running upgrade b20ee72fd9a4 -> 590d89f981c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 590d89f981c0 -> 551c4e6d4a8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 551c4e6d4a8b -> c6e7fc37ad42, empty message
INFO  [alembic.runtime.migration] Running upgrade c6e7fc37ad42 -> 1b7d161d1012, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b7d161d1012 -> 507afb2632cc, empty message
INFO  [alembic.runtime.migration] Running upgrade 507afb2632cc -> 4fac8c8a704c, empty message
INFO  [alembic.runtime.migration] Running upgrade 4fac8c8a704c -> c79c702a1f23, empty message
INFO  [alembic.runtime.migration] Running upgrade c79c702a1f23 -> 5fa68bafae72, empty message
INFO  [alembic.runtime.migration] Running upgrade 5fa68bafae72 -> 4a640c170d02, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a640c170d02 -> 2e2b53afd819, empty message
INFO  [alembic.runtime.migration] Running upgrade 2e2b53afd819 -> 6bbda4685999, empty message
INFO  [alembic.runtime.migration] Running upgrade 6bbda4685999 -> d68a2d971b70, empty message
INFO  [alembic.runtime.migration] Running upgrade d68a2d971b70 -> 0a89c670fc7a, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a89c670fc7a -> 3ebfbaeb76c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 3ebfbaeb76c0 -> 83f4dbe125c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 83f4dbe125c4 -> e505cb517589, empty message
INFO  [alembic.runtime.migration] Running upgrade e505cb517589 -> e83298198ca5, empty message
INFO  [alembic.runtime.migration] Running upgrade e83298198ca5 -> 3a87573bf8a8, empty message
INFO  [alembic.runtime.migration] Running upgrade 3a87573bf8a8 -> a8d8aa307b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade a8d8aa307b8b -> 0b28518684ae, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b28518684ae -> 5e868298fee7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e868298fee7 -> 2d2fc3e826af, empty message
INFO  [alembic.runtime.migration] Running upgrade 2d2fc3e826af -> 0c7f1a48aac9, empty message
INFO  [alembic.runtime.migration] Running upgrade 0c7f1a48aac9 -> 18e934d58f55, empty message
INFO  [alembic.runtime.migration] Running upgrade 18e934d58f55 -> 9e1b06b9df13, empty message
INFO  [alembic.runtime.migration] Running upgrade 9e1b06b9df13 -> d4e4488a0032, empty message
INFO  [alembic.runtime.migration] Running upgrade d4e4488a0032 -> e409f6214b2b, empty message
INFO  [alembic.runtime.migration] Running upgrade e409f6214b2b -> 696e17c13b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 696e17c13b8b -> a8b996f0be40, empty message
INFO  [alembic.runtime.migration] Running upgrade a8b996f0be40 -> 10ad2dbaeccf, empty message
INFO  [alembic.runtime.migration] Running upgrade 10ad2dbaeccf -> 01f808f15b2e, empty message
INFO  [alembic.runtime.migration] Running upgrade 01f808f15b2e -> d29cca963221, empty message
INFO  [alembic.runtime.migration] Running upgrade d29cca963221 -> ba6f13ccbabb, empty message
INFO  [alembic.runtime.migration] Running upgrade ba6f13ccbabb -> 7c39ba4ec38d, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c39ba4ec38d -> 9c976df9b9c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 9c976df9b9c4 -> b9f849432543, empty message
INFO  [alembic.runtime.migration] Running upgrade b9f849432543 -> 6664d75ce3d4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6664d75ce3d4 -> 3c9542fc54e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 3c9542fc54e9 -> 3fa3a648c8e7, empty message
INFO  [alembic.runtime.migration] Running upgrade 3fa3a648c8e7 -> 903ec5f566e8, empty message
INFO  [alembic.runtime.migration] Running upgrade 903ec5f566e8 -> f580030d9beb, empty message
INFO  [alembic.runtime.migration] Running upgrade f580030d9beb -> e3cb44b953f2, empty message
INFO  [alembic.runtime.migration] Running upgrade e3cb44b953f2 -> 75093e7ded27, empty message
INFO  [alembic.runtime.migration] Running upgrade 75093e7ded27 -> 5f191273d067, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f191273d067 -> 7eef64ffb398, empty message
INFO  [alembic.runtime.migration] Running upgrade 7eef64ffb398 -> 235355381f53, empty message
INFO  [alembic.runtime.migration] Running upgrade 235355381f53 -> 628a5438295c, empty message
INFO  [alembic.runtime.migration] Running upgrade 628a5438295c -> 11a35b448f83, empty message
INFO  [alembic.runtime.migration] Running upgrade 11a35b448f83 -> 9081f1a90939, empty message
INFO  [alembic.runtime.migration] Running upgrade 9081f1a90939 -> 91b69dfad2f1, empty message
INFO  [alembic.runtime.migration] Running upgrade 91b69dfad2f1 -> 7744c5c16159, empty message
INFO  [alembic.runtime.migration] Running upgrade 7744c5c16159 -> 14167121af69, empty message
INFO  [alembic.runtime.migration] Running upgrade 14167121af69 -> 6e061eb84167, empty message
INFO  [alembic.runtime.migration] Running upgrade 6e061eb84167 -> e9395fe234a4, empty message
INFO  [alembic.runtime.migration] Running upgrade e9395fe234a4 -> 0809266d08ca, empty message
INFO  [alembic.runtime.migration] Running upgrade 0809266d08ca -> f4b8232fa17e, empty message
INFO  [alembic.runtime.migration] Running upgrade f4b8232fa17e -> dbd80d290f04, empty message
INFO  [alembic.runtime.migration] Running upgrade dbd80d290f04 -> 4e4a759ac4b5, empty message
INFO  [alembic.runtime.migration] Running upgrade 4e4a759ac4b5 -> 30c13ca016e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 30c13ca016e4 -> 541ce53ab6e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 541ce53ab6e9 -> 67c61eead8d2, empty message
INFO  [alembic.runtime.migration] Running upgrade 67c61eead8d2 -> 224fd8963462, empty message
INFO  [alembic.runtime.migration] Running upgrade 224fd8963462 -> 92baf66b268b, empty message
INFO  [alembic.runtime.migration] Running upgrade 92baf66b268b -> 497cfd2a02e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 497cfd2a02e2 -> ea30c0b5b2e3, empty message
INFO  [alembic.runtime.migration] Running upgrade ea30c0b5b2e3 -> bfd7b2302903, empty message
INFO  [alembic.runtime.migration] Running upgrade bfd7b2302903 -> 57ef03f3ac34, empty message
INFO  [alembic.runtime.migration] Running upgrade 57ef03f3ac34 -> dd911f880b75, empty message
INFO  [alembic.runtime.migration] Running upgrade dd911f880b75 -> bd05eac83f5f, empty message
INFO  [alembic.runtime.migration] Running upgrade bd05eac83f5f -> b4146f7d5277, empty message
INFO  [alembic.runtime.migration] Running upgrade b4146f7d5277 -> f939d67374e4, empty message
INFO  [alembic.runtime.migration] Running upgrade f939d67374e4 -> de1b457472e0, empty message
INFO  [alembic.runtime.migration] Running upgrade de1b457472e0 -> ae94fe5c4e9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ae94fe5c4e9f -> 026e7a782ed6, empty message
INFO  [alembic.runtime.migration] Running upgrade 026e7a782ed6 -> 925b93d92809, empty message
INFO  [alembic.runtime.migration] Running upgrade 925b93d92809 -> bdf76f4b65a2, empty message
INFO  [alembic.runtime.migration] Running upgrade bdf76f4b65a2 -> a3a7c518ea70, empty message
INFO  [alembic.runtime.migration] Running upgrade a3a7c518ea70 -> a5e3c6693dc6, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e3c6693dc6 -> bf11ab2f0a7a, empty message
INFO  [alembic.runtime.migration] Running upgrade bf11ab2f0a7a -> 1759f73274ee, empty message
INFO  [alembic.runtime.migration] Running upgrade 1759f73274ee -> 552d735a2f1f, empty message
INFO  [alembic.runtime.migration] Running upgrade 552d735a2f1f -> 5cad8fa84386, empty message
INFO  [alembic.runtime.migration] Running upgrade 5cad8fa84386 -> c31cdf879ee3, empty message
INFO  [alembic.runtime.migration] Running upgrade c31cdf879ee3 -> 659d979b64ce, empty message
INFO  [alembic.runtime.migration] Running upgrade 659d979b64ce -> ce15cf3467b4, empty message
INFO  [alembic.runtime.migration] Running upgrade ce15cf3467b4 -> 0e08145f0499, empty message
INFO  [alembic.runtime.migration] Running upgrade 0e08145f0499 -> 00532ac6d4bc, empty message
INFO  [alembic.runtime.migration] Running upgrade 00532ac6d4bc -> f680032cc361, empty message
INFO  [alembic.runtime.migration] Running upgrade f680032cc361 -> 10a7947fda6b, empty message
INFO  [alembic.runtime.migration] Running upgrade 10a7947fda6b -> 4a7d35941602, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a7d35941602 -> cfc013b6461a, empty message
INFO  [alembic.runtime.migration] Running upgrade cfc013b6461a -> b2d51e4d94c8, empty message
INFO  [alembic.runtime.migration] Running upgrade b2d51e4d94c8 -> 749c2b85d20f, empty message
INFO  [alembic.runtime.migration] Running upgrade 749c2b85d20f -> a5b4dc311a89, empty message
INFO  [alembic.runtime.migration] Running upgrade a5b4dc311a89 -> a3c9a43e41f4, empty message
INFO  [alembic.runtime.migration] Running upgrade a3c9a43e41f4 -> 7128f87af701, empty message
INFO  [alembic.runtime.migration] Running upgrade 7128f87af701 -> 270d598c51e3, empty message
INFO  [alembic.runtime.migration] Running upgrade 270d598c51e3 -> b77ab8c47cc7, empty message
INFO  [alembic.runtime.migration] Running upgrade b77ab8c47cc7 -> a2b95b04d1f7, empty message
INFO  [alembic.runtime.migration] Running upgrade a2b95b04d1f7 -> 63fd3b240583, empty message
INFO  [alembic.runtime.migration] Running upgrade 63fd3b240583 -> 95938a93ea14, empty message
INFO  [alembic.runtime.migration] Running upgrade 95938a93ea14 -> b82bcad9accf, empty message
INFO  [alembic.runtime.migration] Running upgrade b82bcad9accf -> 84471852b610, empty message
INFO  [alembic.runtime.migration] Running upgrade 84471852b610 -> b0e9a389939a, empty message
INFO  [alembic.runtime.migration] Running upgrade b0e9a389939a -> 198c3aca9d8d, empty message
INFO  [alembic.runtime.migration] Running upgrade 198c3aca9d8d -> 58ad4df8583e, empty message
INFO  [alembic.runtime.migration] Running upgrade 58ad4df8583e -> 1abfc9e14d7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 1abfc9e14d7e -> 32b00d06d892, empty message
INFO  [alembic.runtime.migration] Running upgrade 32b00d06d892 -> b17afc77ba83, empty message
INFO  [alembic.runtime.migration] Running upgrade b17afc77ba83 -> 54ca2dbf89c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 54ca2dbf89c0 -> eef0c404b531, empty message
INFO  [alembic.runtime.migration] Running upgrade eef0c404b531 -> 84dec6c29c48, empty message
INFO  [alembic.runtime.migration] Running upgrade 84dec6c29c48 -> d0f197979bd9, empty message
INFO  [alembic.runtime.migration] Running upgrade d0f197979bd9 -> 9dc16e591f88, empty message
INFO  [alembic.runtime.migration] Running upgrade 9dc16e591f88 -> ac41029fb329, empty message
INFO  [alembic.runtime.migration] Running upgrade ac41029fb329 -> d1edb3cadec8, empty message
INFO  [alembic.runtime.migration] Running upgrade d1edb3cadec8 -> 623662ea0e7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 623662ea0e7e -> 56c790ec8ab4, empty message
INFO  [alembic.runtime.migration] Running upgrade 56c790ec8ab4 -> c0d91ff18f77, empty message
INFO  [alembic.runtime.migration] Running upgrade c0d91ff18f77 -> 780a8344914b, empty message
INFO  [alembic.runtime.migration] Running upgrade 780a8344914b -> a20aeb9b0eac, empty message
INFO  [alembic.runtime.migration] Running upgrade a20aeb9b0eac -> 0af2c2e286a7, empty message
INFO  [alembic.runtime.migration] Running upgrade 0af2c2e286a7 -> 1919f1859215, empty message
INFO  [alembic.runtime.migration] Running upgrade 1919f1859215 -> f66ca777f409, empty message
INFO  [alembic.runtime.migration] Running upgrade f66ca777f409 -> 7c0dbd378cdb, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c0dbd378cdb -> e99989e6ad56, empty message
INFO  [alembic.runtime.migration] Running upgrade e99989e6ad56 -> 1b54995bc086, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b54995bc086 -> 2779eb90c6c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 2779eb90c6c4 -> 74906d31d994, empty message
INFO  [alembic.runtime.migration] Running upgrade 74906d31d994 -> 85d0655d42c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 85d0655d42c0 -> de7aa5280210, empty message
INFO  [alembic.runtime.migration] Running upgrade de7aa5280210 -> e831a883153a, empty message
INFO  [alembic.runtime.migration] Running upgrade e831a883153a -> d1236c4dff71, empty message
INFO  [alembic.runtime.migration] Running upgrade d1236c4dff71 -> 94f14eb0fe5b, empty message
INFO  [alembic.runtime.migration] Running upgrade 94f14eb0fe5b -> 9d6adad83936, empty message
INFO  [alembic.runtime.migration] Running upgrade 9d6adad83936 -> f398b261d9c6, empty message
INFO  [alembic.runtime.migration] Running upgrade f398b261d9c6 -> 517b79c56088, empty message
INFO  [alembic.runtime.migration] Running upgrade 517b79c56088 -> 48b991e9de06, empty message
INFO  [alembic.runtime.migration] Running upgrade 48b991e9de06 -> e11c3dd48a6f, empty message
INFO  [alembic.runtime.migration] Running upgrade e11c3dd48a6f -> 4912f3bd5ba2, empty message
INFO  [alembic.runtime.migration] Running upgrade 4912f3bd5ba2 -> f5133dc851ee, empty message
INFO  [alembic.runtime.migration] Running upgrade f5133dc851ee -> 5c77d685df87, empty message
INFO  [alembic.runtime.migration] Running upgrade 5c77d685df87 -> 6cc7f073b358, empty message
INFO  [alembic.runtime.migration] Running upgrade 6cc7f073b358 -> 68e2f38e33f4, empty message
INFO  [alembic.runtime.migration] Running upgrade 68e2f38e33f4 -> fc2eb1d7e4fc, empty message
INFO  [alembic.runtime.migration] Running upgrade fc2eb1d7e4fc -> a5e643d562c9, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e643d562c9 -> 29ea13ed76f9, empty message
INFO  [alembic.runtime.migration] Running upgrade 29ea13ed76f9 -> 8e70205a5308, empty message
INFO  [alembic.runtime.migration] Running upgrade 8e70205a5308 -> f3f19998b755, empty message
INFO  [alembic.runtime.migration] Running upgrade f3f19998b755 -> c31a081eab74, empty message
INFO  [alembic.runtime.migration] Running upgrade c31a081eab74 -> 78403c7b8089, empty message
INFO  [alembic.runtime.migration] Running upgrade 78403c7b8089 -> 5662122eac21, empty message
INFO  [alembic.runtime.migration] Running upgrade 5662122eac21 -> 20c738810b1b, empty message
INFO  [alembic.runtime.migration] Running upgrade 20c738810b1b -> dfee471558bd, empty message
INFO  [alembic.runtime.migration] Running upgrade dfee471558bd -> 05e3af59929a, empty message
INFO  [alembic.runtime.migration] Running upgrade 05e3af59929a -> c3470e2d3224, empty message
INFO  [alembic.runtime.migration] Running upgrade c3470e2d3224 -> ffa75d04e6ef, empty message
INFO  [alembic.runtime.migration] Running upgrade ffa75d04e6ef -> 9014cca7097c, empty message
INFO  [alembic.runtime.migration] Running upgrade 9014cca7097c -> d4392342465f, empty message
INFO  [alembic.runtime.migration] Running upgrade d4392342465f -> 424808e1fe49, empty message
INFO  [alembic.runtime.migration] Running upgrade 424808e1fe49 -> 916a5257d18c, empty message
INFO  [alembic.runtime.migration] Running upgrade 916a5257d18c -> 4d3f91ddf3e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 4d3f91ddf3e9 -> d8c55e79da54, empty message
INFO  [alembic.runtime.migration] Running upgrade d8c55e79da54 -> cf1e8c1bc737, empty message
INFO  [alembic.runtime.migration] Running upgrade cf1e8c1bc737 -> 7a105bfc0cd0, empty message
INFO  [alembic.runtime.migration] Running upgrade 7a105bfc0cd0 -> bc75acacc98e, empty message
INFO  [alembic.runtime.migration] Running upgrade bc75acacc98e -> b8b4f9598240, empty message
INFO  [alembic.runtime.migration] Running upgrade b8b4f9598240 -> 5ee767807344, empty message
INFO  [alembic.runtime.migration] Running upgrade 5ee767807344 -> 4913cb3f5a05, empty message
INFO  [alembic.runtime.migration] Running upgrade 4913cb3f5a05 -> 0b1c9ea11aef, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b1c9ea11aef -> 2fbcad5527d7, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fbcad5527d7 -> d750d578b068, empty message
INFO  [alembic.runtime.migration] Running upgrade d750d578b068 -> 2f1b3c759773, empty message
INFO  [alembic.runtime.migration] Running upgrade 2f1b3c759773 -> 99d9e329b27f, empty message
INFO  [alembic.runtime.migration] Running upgrade 99d9e329b27f -> a06066e3fbeb, empty message
INFO  [alembic.runtime.migration] Running upgrade a06066e3fbeb -> d67eab226ecd, empty message
INFO  [alembic.runtime.migration] Running upgrade d67eab226ecd -> bbedc353f90c, empty message
INFO  [alembic.runtime.migration] Running upgrade bbedc353f90c -> 0b9150eb309d, Increase message_id length manually
INFO  [alembic.runtime.migration] Running upgrade 0b9150eb309d -> 6204e57b4bc4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6204e57b4bc4 -> 37feaba7c45d, empty message
INFO  [alembic.runtime.migration] Running upgrade 37feaba7c45d -> ff6c04869029, empty message
INFO  [alembic.runtime.migration] Running upgrade ff6c04869029 -> fdb02bd105a8, empty message
INFO  [alembic.runtime.migration] Running upgrade fdb02bd105a8 -> dd278f96ca83, empty message
INFO  [alembic.runtime.migration] Running upgrade dd278f96ca83 -> 1076b5795b08, empty message
INFO  [alembic.runtime.migration] Running upgrade 1076b5795b08 -> 5639ad89ee50, empty message
INFO  [alembic.runtime.migration] Running upgrade 5639ad89ee50 -> 11ba83e2dd71, empty message
INFO  [alembic.runtime.migration] Running upgrade 11ba83e2dd71 -> ccbfb61eda0d, empty message
INFO  [alembic.runtime.migration] Running upgrade ccbfb61eda0d -> a5013ff0a00a, empty message
INFO  [alembic.runtime.migration] Running upgrade a5013ff0a00a -> e6e8e12f5a13, empty message
INFO  [alembic.runtime.migration] Running upgrade e6e8e12f5a13 -> 9031c9e28510, empty message
INFO  [alembic.runtime.migration] Running upgrade 9031c9e28510 -> b8fd175c084a, empty message
INFO  [alembic.runtime.migration] Running upgrade b8fd175c084a -> e7d7ebcea26c, empty message
INFO  [alembic.runtime.migration] Running upgrade e7d7ebcea26c -> d0ccd9d7ac0c, empty message
INFO  [alembic.runtime.migration] Running upgrade d0ccd9d7ac0c -> 4b483a762fed, empty message
INFO  [alembic.runtime.migration] Running upgrade 4b483a762fed -> ad467baf7ec8, empty message
INFO  [alembic.runtime.migration] Running upgrade ad467baf7ec8 -> d8a3dfe674f2, empty message
INFO  [alembic.runtime.migration] Running upgrade d8a3dfe674f2 -> 3d05479d0d11, empty message
INFO  [alembic.runtime.migration] Running upgrade 3d05479d0d11 -> 753d2ed92d41, empty message
INFO  [alembic.runtime.migration] Running upgrade 753d2ed92d41 -> 698424c429e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 698424c429e9 -> 07b870d7cc86, empty message
INFO  [alembic.runtime.migration] Running upgrade 07b870d7cc86 -> 9282e982bc05, Add block_behaviour setting for user
INFO  [alembic.runtime.migration] Running upgrade 9282e982bc05 -> 5047fcbd57c7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5047fcbd57c7 -> 4729b7096d12, empty message
INFO  [alembic.runtime.migration] Running upgrade 4729b7096d12 -> b500363567e3, Create admin audit log
INFO  [alembic.runtime.migration] Running upgrade b500363567e3 -> 28b9b14c9664, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 28b9b14c9664 -> 0aaad1740797, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 0aaad1740797 -> e866ad0e78e1, Add partner tables
INFO  [alembic.runtime.migration] Running upgrade e866ad0e78e1 -> 088f23324464, add flags to the user model
INFO  [alembic.runtime.migration] Running upgrade 088f23324464 -> 2b1d3cd93e4b, update partner_api_token token length
INFO  [alembic.runtime.migration] Running upgrade 2b1d3cd93e4b -> 82d3c7109ffb, partner_user and partner_subscription
INFO  [alembic.runtime.migration] Running upgrade 82d3c7109ffb -> 36646e5dc6d9, make external_user_id non nullable
INFO  [alembic.runtime.migration] Running upgrade 36646e5dc6d9 -> a7bcb872c12a, Add alias transfer token expiration
INFO  [alembic.runtime.migration] Running upgrade a7bcb872c12a -> 673a074e4215, empty message
INFO  [alembic.runtime.migration] Running upgrade 673a074e4215 -> d1fb679f7eec, Add sudo expiration for ApiKeys
INFO  [alembic.runtime.migration] Running upgrade d1fb679f7eec -> bfebc2d5c719, Add state to job
INFO  [alembic.runtime.migration] Running upgrade bfebc2d5c719 -> 516c21ea7d87, empty message
INFO  [alembic.runtime.migration] Running upgrade 516c21ea7d87 -> bd7d032087b2, empty message
INFO  [alembic.runtime.migration] Running upgrade bd7d032087b2 -> b0101a66bb77, Add unsubscribe behaviour
INFO  [alembic.runtime.migration] Running upgrade b0101a66bb77 -> 89081a00fc7d, default_unsub_behaviour
INFO  [alembic.runtime.migration] Running upgrade 89081a00fc7d -> c66f2c5b6cb1, empty message
INFO  [alembic.runtime.migration] Running upgrade c66f2c5b6cb1 -> 9cc0f0712b29, Add api to cookie token
INFO  [alembic.runtime.migration] Running upgrade 9cc0f0712b29 -> bd95b2b4217f, Updated recovery code string length
INFO  [alembic.runtime.migration] Running upgrade bd95b2b4217f -> 2c2093c82bc0, empty message
INFO  [alembic.runtime.migration] Running upgrade 2c2093c82bc0 -> 5f4a5625da66, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f4a5625da66 -> 893c0d18475f, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> bc496c0a0279, empty message
INFO  [alembic.runtime.migration] Running upgrade bc496c0a0279 -> 2d89315ac650, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> 01e2997e90d3, empty message
INFO  [alembic.runtime.migration] Running upgrade 01e2997e90d3, 2d89315ac650 -> 2634b41f54db, empty message
INFO  [alembic.runtime.migration] Running upgrade 2634b41f54db -> 01827104004b, empty message
INFO  [alembic.runtime.migration] Running upgrade 01827104004b -> 0a5701a4f5e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a5701a4f5e4 -> ec7fdde8da9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ec7fdde8da9f -> 46ecb648a47e, empty message
INFO  [alembic.runtime.migration] Running upgrade 46ecb648a47e -> 4bc54632d9aa, empty message
INFO  [alembic.runtime.migration] Running upgrade 4bc54632d9aa -> 818b0a956205, empty message
INFO  [alembic.runtime.migration] Running upgrade 818b0a956205 -> 52510a633d6f, empty message
INFO  [alembic.runtime.migration] Running upgrade 52510a633d6f -> fa2f19bb4e5a, empty message
INFO  [alembic.runtime.migration] Running upgrade fa2f19bb4e5a -> 06a9a7133445, Create sync_event table
INFO  [alembic.runtime.migration] Running upgrade 06a9a7133445 -> d608b8e48082, empty message
INFO  [alembic.runtime.migration] Running upgrade d608b8e48082 -> 56d08955fcab, add retry count to sync event
INFO  [alembic.runtime.migration] Running upgrade 56d08955fcab -> 1c14339aae90, empty message
INFO  [alembic.runtime.migration] Running upgrade 1c14339aae90 -> 2441b7ff5da9, Custom Domain partner id
INFO  [alembic.runtime.migration] Running upgrade 2441b7ff5da9 -> 88dd7a0abf54, contact.flags and custom_domain.pending_deletion
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

---

## 13. Appendix B — Consolidated, ordered reproduction script

The single script below performs **every** step of the investigation in the exact order required, from a clean host with only Docker available. It creates the container id file first (so teardown can read it), installs the environment fixes, provisions Postgres/Redis, installs and starts the **MailHog** canonical sink, creates `.env` **before** any handler import, migrates and seeds, starts the handler, builds the fixed payload, sends the three success runs + one failure + three new-sender runs, snapshots the database, derives the reverse-alias dynamically, and finally tears everything down. It never prints the Postgres password. Run it end-to-end with `bash`.

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- 0. workspace + container id file (written FIRST, read by teardown) ---
INV_DIR=/tmp/blitzy_inv_06d10137
mkdir -p "$INV_DIR"
IMG="ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0"
INV_NAME="sl_inv_06d10137_$(date +%s)"
CID=$(docker run -d --name "$INV_NAME" \
        --label blitzy-investigation=app_2cd6ee777f8c \
        --entrypoint bash "$IMG" -lc "sleep infinity")
echo "$CID" > "$INV_DIR/container_id.txt"

dex() { docker exec "$CID" bash -lc "$1"; }

# --- 1. environment fixes (venv only; NO product file touched) ---
dex '/app/venv/bin/pip uninstall -y google-re2 && /app/venv/bin/pip install --only-binary :all: pyre2==0.3.10'
dex 'apt-get update && apt-get install -y swaks'

# --- 2. MailHog canonical sink (SMTP :1025, API/UI :1080) ---
dex 'wget -q -O /usr/local/bin/MailHog https://github.com/mailhog/MailHog/releases/download/v1.0.1/MailHog_linux_amd64 && chmod +x /usr/local/bin/MailHog'
docker exec -d "$CID" bash -lc 'export MH_SMTP_BIND_ADDR=127.0.0.1:1025 MH_API_BIND_ADDR=127.0.0.1:1080 MH_UI_BIND_ADDR=127.0.0.1:1080 MH_STORAGE=memory; nohup /usr/local/bin/MailHog >/tmp/mailhog.log 2>&1 &'

# --- 3. Postgres (move to 15432, provision role/db) + Redis ---
dex 'pg_conftool 15 main set port 15432 && pg_ctlcluster 15 main start'
dex '
  PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"
  su postgres -c "psql -p 15432 -v ON_ERROR_STOP=1 -c \"CREATE ROLE myuser SUPERUSER LOGIN PASSWORD '"'"'${PGPASS}'"'"';\""
  su postgres -c "psql -p 15432 -v ON_ERROR_STOP=1 -c \"CREATE DATABASE simplelogin OWNER myuser;\""
'
dex 'redis-server --daemonize yes --port 6379'

# --- 4. .env (four deltas) — created BEFORE any handler import ---
dex 'cd /app && cp example.env .env && \
  sed -i "s|^NOT_SEND_EMAIL=true|# NOT_SEND_EMAIL=true|" .env && \
  sed -i "s|^# POSTFIX_SERVER=my-postfix.com|POSTFIX_SERVER=localhost|" .env && \
  sed -i "s|@localhost:5432/simplelogin|@localhost:15432/simplelogin|" .env && \
  sed -i "s|^# POSTFIX_PORT=1025|POSTFIX_PORT=1025|" .env'

# --- 5. migrate (creates pg_trgm on clean DB) + seed ---
dex 'cd /app && CONFIG=/app/.env /app/venv/bin/alembic upgrade head'
dex 'cd /app && CONFIG=/app/.env FLASK_APP=wsgi:app /app/venv/bin/flask dummy-data'

# --- 6. start canonical handler on :20381 ---
docker exec -d "$CID" bash -lc 'cd /app && nohup env CONFIG=/app/.env /app/venv/bin/python email_handler.py >/tmp/handler.log 2>&1 &'
sleep 6

# --- 7. fixed byte-identical payload ---
dex 'printf "Date: Mon, 13 Jul 2026 12:00:00 +0000\r\nTo: e1@sl.local\r\nFrom: hey@google.com\r\nSubject: Blitzy fixed forward probe\r\nMessage-ID: <blitzy-fixed-probe-2cd6ee777f8c@test.local>\r\nContent-Type: text/plain; charset=UTF-8\r\nContent-Transfer-Encoding: 7bit\r\n\r\nFixed byte-identical body for run-to-run replay. Do not change.\r\n" > /tmp/payload.eml'

# --- 8. three identical success sends + one failure ---
for r in 1 2 3; do
  dex 'swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 --data - < /tmp/payload.eml'
  sleep 1
done
dex 'swaks --to doesnotexist@sl.local --from hey@google.com --server 127.0.0.1:20381 --data - < /tmp/payload.eml'

# --- 9. three NEW-SENDER runs (Q3 suffix distribution) ---
for s in alice bob carol; do
  dex "printf 'Date: Mon, 13 Jul 2026 12:30:00 +0000\r\nTo: e1@sl.local\r\nFrom: ${s}@example.org\r\nSubject: New-sender reverse-alias probe\r\nMessage-ID: <newsender-${s}-2cd6ee777f8c@test.local>\r\nContent-Type: text/plain; charset=UTF-8\r\nContent-Transfer-Encoding: 7bit\r\n\r\nProbe for reverse-alias suffix distribution.\r\n' > /tmp/ns_${s}.eml"
  dex "swaks --to e1@sl.local --from ${s}@example.org --server 127.0.0.1:20381 --data - < /tmp/ns_${s}.eml"
  sleep 1
done

# --- 10. DB snapshots + dynamic reverse-alias derivation ---
dex 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "select id,alias_id,website_email,reply_email,created_at from contact order by id;"'
dex 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "select id,contact_id,user_id,mailbox_id,message_id,sl_message_id,created_at from email_log order by id;"'
dex 'PGPASS="$(grep -m1 "^DB_URI=" /app/example.env | sed -E "s#.*//[^:]+:([^@]+)@.*#\1#")"; PGPASSWORD="$PGPASS" psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "select reply_email from contact where website_email='"'"'hey@google.com'"'"' and alias_id=5;"'

# --- 11. delivered headers from MailHog (canonical sink) ---
dex 'wget -qO- http://127.0.0.1:1080/api/v2/messages | /app/venv/bin/python -c "import json,sys; d=json.load(sys.stdin); print(\"delivered:\", d[\"total\"])"'

# --- 12. teardown (container removal takes services with it) ---
CID="$(cat "$INV_DIR/container_id.txt")"
test -n "$CID" && docker rm -f "$CID"
case "$(realpath "$INV_DIR")" in /tmp/blitzy_inv_06d10137) rm -rf "$INV_DIR";; *) echo "refuse: $INV_DIR";; esac
```

This script is the authoritative, ordered record; §2–§11 above narrate and evidence each numbered step with its captured output.
