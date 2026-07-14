# SimpleLogin Reply-Resolution Investigation — How an Inbound Reply Resolves to a Contact and Forwarding Destination (and Why It Can Forward to the Wrong User)

> **Scope.** Read-only, run-first root-cause investigation. This Markdown file is the *only* persistent artifact produced. No product, source, model, migration, test, configuration, or manifest file was modified, and **no remediation** (no `UNIQUE` constraint, no `ORDER BY`, no atomic generator, no lock/retry) was performed. All temporary observation scripts live **outside** the repository (in `/tmp`) and were removed afterward, with explicit absence proof in §13.
>
> **Methodology.** Every behavioral claim is backed by the *actual, complete, unedited* output of a command that exercised the **canonical** inbound reply path — `email_handler.handle_reply(envelope, msg, rcpt_to)` (`email_handler.py:966`), and, for the routing claim, the real routing hub `email_handler.handle(envelope, msg)` (`email_handler.py:1945`) — inside the canonical Docker runtime (Python 3.10 + PostgreSQL 15 + Redis). Factual claims are grounded in a specific `file:line`. Values obtained by bypassing the entry point (direct ORM/helper calls) are labeled **`[non-canonical]`**; the per-alias spoof-check-disabled path is labeled **`[non-canonical]` fallback**; statements not directly observed are labeled **`[inferred]`**.
>
> **Evidence fidelity note.** Each command and its output are reproduced from the exact files captured at runtime (saved with combined `stdout`+`stderr`, i.e. `2>&1` — no stream is suppressed). The **only** transformation applied when embedding them here is the removal of *trailing* whitespace at line ends (present solely in `psql`'s column-padding) and the normalization of the final newline, so that `git diff --check` reports no warnings (§7.4, §13). No value, identifier, count, log line, ordering, or structural content was altered. Every behavioral section shows the command that produced the output and was confirmed stable across **≥2 runs**; where run 1 is shown, run 2 is **byte-identical modulo** only the intentionally varied Message-ID prefix / random reverse-alias token / logger timestamps+PID, proven per section by a reproducible `maskdoc`/`diff`/`sha256sum` comparison with matching SHA256 and `exit=0` (§4.2, §4.3, §8.1–§8.3, §8.8, §8.10, §10).

---
## 1. Lead Answer (Executive Summary)

**How an inbound reply resolves to a contact and a forwarding destination.** When a user replies to a *reverse-alias*, the inbound SMTP recipient (`rcpt_to`) is taken verbatim as the reply address: `reply_email = rcpt_to` (`email_handler.py:972`). It is gated against the service domain — `if not reply_email.endswith(EMAIL_DOMAIN)` (`email_handler.py:977`), falling back to an `SLDomain` lookup (`email_handler.py:978`) and returning `status.E501` if neither matches (`email_handler.py:980-981`). It is then normalized: `reply_email = normalize_reply_email(reply_email)` (`email_handler.py:984`). The normalized value resolves a **single** `Contact`: `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:986`). The **forwarding destination is derived entirely from that resolved contact**: `alias = contact.alias` (`email_handler.py:994`), `user = alias.user` (`email_handler.py:1004`), and the sender mailbox is chosen by `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:1019`). On the authorized path the resolution is persisted as `EmailLog.create(contact_id=contact.id, alias_id=contact.alias_id, is_reply=True, user_id=contact.user_id, mailbox_id=mailbox.id, ...)` (`email_handler.py:1042-1050`).

**Why it *can* forward to the wrong user.** The lookup `Contact.get_by(reply_email=...)` delegates to the shared `ModelMixin.get_by()`, whose entire body is `return Session.query(cls).filter_by(**kw).first()` — a `.first()` with **no `ORDER BY`** (`app/models.py:82-84`). The `reply_email` column carries **no `UNIQUE` constraint**: the only unique constraint on `Contact` is `uq_contact` on `(alias_id, website_email)` (`app/models.py:1875`); `reply_email` is merely `index=True` (`app/models.py:1899`), and the migration that created its index sets `unique=False` (`migrations/versions/2021_071310_78403c7b8089_.py:22`). Generation-time uniqueness is only a best-effort, non-atomic time-of-check-to-time-of-use (TOCTOU) guard, `available_sl_email()` (`app/models.py:1425-1432`), which non-generator write paths (a direct `Contact.create(...)`) bypass entirely. Consequently, **two `Contact` rows owned by different users can share one `reply_email`**, and on such a multi-row match `.first()` returns one row **without any application-specified order**. Because the alias and user are derived from that row, the reply is routed to — and, on the authorized path, logged against — **whichever contact the database returned**, not necessarily the alias owner whose mailbox actually sent the reply.

**What the runtime evidence actually shows (and the important distinction the default access control enforces).** Seeding two contacts that share one `reply_email` on aliases owned by different users, then driving the *identical* canonical input against those same persisted rows, the resolved contact — and therefore the alias, user, and mailbox derived from it — is determined by which row `.first()` returns, which in turn tracks the contacts' **insertion order** under the active query plan (§7):

- Insertion order **A-then-B** → **100/100** events (× 2 runs) resolved to the first-inserted contact, `user_A` (the correct owner of the sending mailbox). Result: **E200**, `EmailLog.user_id = 1`, **100** outbound `SendRequest`s to `a-dist@nowhere.net`.
- Insertion order **B-then-A**, with the **identical** `mail_from` (user_A's mailbox) and `rcpt_to` → `.first()` resolves `user_B`'s contact — **a different user than the sending mailbox owner**. What happens next depends on the alias's spoof-check setting:
  - **Default control (`disable_email_spoofing_check = False`, the out-of-the-box behavior)** → `get_mailbox_from_mail_from()` returns `None` (user_A's mailbox is not authorized for user_B's alias), so `handle_reply()` calls `handle_unknown_mailbox()` and returns **`status.E214`** (`email_handler.py:1032-1034`). The reply is **rejected before any `EmailLog` is created and before any forward**; the only outbound messages are **rate-limited `reverse_alias_unknown_mailbox` alerts to `user_B`** (`userb@mailbox.test`) warning that someone tried to use their alias — **4** emails on the first run over the fresh dataset, then **0** on the repeat once the per-recipient/24h cap is reached (a **cross-user disclosure**, since `user_B` is *not* the sender). This is an **access-control boundary**, not a wrong-user delivery. Observed **100/100** (× 2 runs): `code = '250 SL E214 Unauthorized for using reverse alias'`, `EmailLog = None`.
  - **`[non-canonical]` fallback (`disable_email_spoofing_check = True`, a non-default per-alias flag)** → the spoof check is skipped (`email_handler.py:1023`), the sender mailbox falls back to `alias.mailbox`, and the reply is **selected, logged, and a send request is enqueued under the wrong user**: **E200**, `EmailLog.user_id = 2 (user_B)`, **100** outbound `SendRequest`s to external target `b-dist@nowhere.net`. Observed **100/100** (× 2 runs). This is the condition under which a reply is actually *forwarded* under the wrong user.

**Two clarifications the evidence makes precise.** (1) *Application acceptance is not confirmed external delivery.* The harness runs with `NOT_SEND_EMAIL=true`, so `mail_sender.send()` logs and returns success **without** calling the SMTP transport (`app/mail_sender.py:130-136`); an observed `'250 Message accepted for delivery'` and a stored `SendRequest` prove *selection, logging, and enqueue*, not a confirmed outbound SMTP hand-off. (2) *The resolved row is not attributable to a specified ordering.* The application specifies no `ORDER BY`; the row the database returns depends on the active plan and physical layout (§7.4). Within each fixed layout the result was **stable** (100/100 across 2 runs, SHA256-verified byte-identical on the resolution dimension), not random per call; that a *different* plan or layout could return the other row is **`[inferred]`** from the absence of an `ORDER BY` plus the official SQLAlchemy/PostgreSQL semantics cited in §4.1 and §9.

Full outputs and commands are in §3–§8; the complete causal chain is in §6; the cross-event distribution (every seed and every run) is in §7.

---
## 2. Environment & Methodology

### 2.1 Canonical runtime (Python 3.10 + PostgreSQL 15 + Redis)

All observations were produced inside the canonical Docker container `sl_app` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), which contains the SimpleLogin repository at commit `2cd6ee777f8c` — the commit the assigned filename encodes (branch grounding in §2.3). The command below prints each fact under its own explicit label so that the shown output is exactly what the command emits (each separator line — `--- python (system) ---`, `--- postgres ---`, `--- redis ---` — is produced by an `echo` in the command itself).

**Command:**

```
docker exec sl_app bash -lc '
echo "--- python (system) ---"; python --version 2>&1
. /app/venv/bin/activate
echo "--- python (venv) ---"; python --version 2>&1
echo "--- postgres ---"; pg_isready 2>&1; su postgres -c "psql -tAc \"select version();\"" 2>&1 | head -1
echo "--- redis ---"; redis-cli ping 2>&1
'
```

**Output (complete, unedited; stable across run 1 and run 2):**

```
--- python (system) ---
Python 3.10.18
--- python (venv) ---
Python 3.10.18
--- postgres ---
/var/run/postgresql:5432 - accepting connections
PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
--- redis ---
PONG
```

This matches the AAP's canonical configuration: **Python 3.10.18**, **PostgreSQL 15.13**, and **Redis** (`PONG`). The venv Python equals the system Python (3.10.18), so activation does not change the interpreter version.

### 2.2 Canonical configuration values (and the dotenv precedence that governs `DB_URI`)

The reply path depends on `EMAIL_DOMAIN` (the reply-domain gate at `email_handler.py:977`) and `DB_URI` (the datastore). The runtime recipe `/tmp/sl_env.sh` (sourced from `tests/test.env` during image build) exports these into the environment *before* the app imports its config.

**Command:**

```
docker exec sl_app bash -lc '
cd /app
echo "--- effective runtime env (from /tmp/sl_env.sh) ---"
set -a && . /tmp/sl_env.sh && set +a
echo "CONFIG=$CONFIG"; echo "DB_URI=$DB_URI"; echo "EMAIL_DOMAIN=$EMAIL_DOMAIN"; echo "NOT_SEND_EMAIL=$NOT_SEND_EMAIL"
echo "--- source lines: tests/test.env ---"
grep -n "^EMAIL_DOMAIN\|^DB_URI" tests/test.env
echo "--- source line: example.env ---"
grep -n "^EMAIL_DOMAIN" example.env
echo "--- dotenv precedence grounding: app/config.py load_dotenv call ---"
grep -n "load_dotenv" app/config.py
echo "--- NOT_SEND_EMAIL grounding: app/config.py ---"
grep -n "NOT_SEND_EMAIL" app/config.py
'
```

**Output (complete, unedited; stable across run 1 and run 2):**

```
--- effective runtime env (from /tmp/sl_env.sh) ---
CONFIG=/app/tests/test.env
DB_URI=postgresql://test:test@localhost:5432/test
EMAIL_DOMAIN=sl.local
NOT_SEND_EMAIL=true
--- source lines: tests/test.env ---
8:EMAIL_DOMAIN=sl.local
17:DB_URI=postgresql://test:test@localhost:15432/test
--- source line: example.env ---
22:EMAIL_DOMAIN=sl.local
--- dotenv precedence grounding: app/config.py load_dotenv call ---
9:from dotenv import load_dotenv
69:    load_dotenv(get_abs_path(config_file))
71:    load_dotenv()
--- NOT_SEND_EMAIL grounding: app/config.py ---
91:NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ
```

- `EMAIL_DOMAIN=sl.local` is declared at `tests/test.env:8` and `example.env:22`, and is the effective runtime value.
- **`DB_URI` precedence (grounded, not asserted):** `tests/test.env:17` declares port **15432**, but the effective runtime value is `...@localhost:5432/test`. The reason is grounded in source: `app/config.py` calls `load_dotenv(get_abs_path(config_file))` (`app/config.py:69`) or `load_dotenv()` (`app/config.py:71`). Python-dotenv's `load_dotenv()` does **not** override variables already present in the environment (its `override` parameter defaults to `False`); because `/tmp/sl_env.sh` already exported `DB_URI=...5432`, the exported value wins and the app connects directly to the in-container PostgreSQL on `5432` (no `socat` bridge needed). This is a harness/runtime detail, not a code change. The `5432` value is a member of the disposable-DB allowlist the harnesses enforce before any write (§2.7, §7).
- `NOT_SEND_EMAIL=true` is in effect; it is read at `app/config.py:91` as `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ`. Its consequence for "delivery" claims is made precise throughout (§4.2, §7): success codes prove application acceptance and enqueue, not confirmed SMTP hand-off.

### 2.3 Runtime branch / HEAD confirmation (and honest disclosure of the detached, setup-dirty container)

The mandated filename `app_2cd6ee777f8c.md` is derived from the **source branch** `app_2cd6ee777f8c`. The runtime container's `/app` checkout is **detached** at the corresponding commit (it is not on a named branch), so the branch name is grounded from the commit and the destination repository — not asserted from a container symbolic ref. Both are shown, and the container's setup-time dirtiness is disclosed.

**Command:**

```
echo "=== CONTAINER /app checkout (canonical runtime source) ==="
docker exec sl_app bash -lc 'cd /app; echo "HEAD commit:"; git rev-parse HEAD; echo "symbolic-ref (branch):"; git symbolic-ref -q HEAD || echo "(none: DETACHED HEAD)"; echo "describe:"; git describe --all --always 2>&1; echo "dirty (git status --porcelain):"; git status --porcelain'
echo ""
echo "=== DESTINATION repo (host working tree; where the deliverable lives) ==="
cd /tmp/blitzy/app/blitzy-3fc9b061-bac4-42a9-b984-8eaab62d81e6_193781
echo "branch:"; git rev-parse --abbrev-ref HEAD
echo "HEAD commit:"; git rev-parse HEAD
```

**Output (complete, unedited; stable across run 1 and run 2):**

```
=== CONTAINER /app checkout (canonical runtime source) ===
HEAD commit:
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
symbolic-ref (branch):
(none: DETACHED HEAD)
describe:
tags/v4.53.2-6-g2cd6ee77
dirty (git status --porcelain):
 M app/spamassassin_utils.py
 M local_data/jwtRS256.key
 M local_data/jwtRS256.key.pub
 M local_data/test_words.txt
 M static/package-lock.json

=== DESTINATION repo (host working tree; where the deliverable lives) ===
branch:
blitzy-3fc9b061-bac4-42a9-b984-8eaab62d81e6
HEAD commit:
864d5041fd239b020b0a73ba87a60a3a8d577564
```

- **Container `/app` (canonical runtime source):** `HEAD = 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`, **DETACHED HEAD** (no symbolic ref), `git describe = tags/v4.53.2-6-g2cd6ee77`. It is **dirty in exactly five setup-generated files** — `app/spamassassin_utils.py`, `local_data/jwtRS256.key`, `local_data/jwtRS256.key.pub`, `local_data/test_words.txt`, `static/package-lock.json` — none of which is on the reply-resolution path or touched by this investigation. This is a property of the pre-built image's bring-up (documented in the setup notes), disclosed here for reproducibility.
- **Destination repository (host working tree; where this deliverable lives):** branch `blitzy-3fc9b061-bac4-42a9-b984-8eaab62d81e6`, `HEAD = 864d5041...`. **All repository-integrity claims in this document are scoped to this destination repository** (§13): the sole change here is the creation of this one Markdown file.
- **Authoring-time snapshot (why this HEAD is not the final commit).** The `HEAD = 864d5041…` shown above is the value captured **while this section's command was run** — the commit that first added this deliverable. The document is then revised across further commits, each of which advances `HEAD` (the rewrite commit `1639aadf`, then the QA-fix commit `bcc20d65`, then the commit carrying the present edits). The hash printed here is therefore necessarily an **authoring-time snapshot**, and the document's **own final commit hash is not self-citable** — a file cannot embed the hash of the commit that will contain it. This does not weaken the integrity invariant: **every** commit that has ever touched this path changes **only** this one Markdown file (verified name-only in §13), so "the sole change is this one file" holds both at authoring time and after each commit.

### 2.4 How the app was booted (mirroring `tests/conftest.py`)

Each temporary harness boots a *real* Flask app exactly as the canonical test fixture does, then works inside `app.app_context()`:

1. `CONFIG` points to `tests/test.env` (via `/tmp/sl_env.sh`) and is exported **before** app modules are imported (matching `tests/conftest.py`), so `app/config.py`'s `load_dotenv` sees the canonical values.
2. `from server import create_app` (`server.py:139` — `def create_app() -> Flask:`), then `app = create_app()`.
3. `CREATE EXTENSION IF NOT EXISTS pg_trgm` is executed idempotently (the `pg_trgm` extension backs SimpleLogin's trigram indexes). The harnesses **never** `DROP` the extension (contrast with the conftest teardown), avoiding privileged destructive DDL (§2.7 safety).
4. `from init_app import add_sl_domains, add_proton_partner`; `add_sl_domains()` seeds the `SLDomain` rows the domain gate needs (including `sl.local` — visible in the boot logs of every run, e.g. `Add sl.local to SL domain`); `add_proton_partner()` seeds the Proton partner.
5. The ORM session is `from app.db import Session, engine`.
6. `mail_sender.store_emails_instead_of_sending(True)` (`app/mail_sender.py`) captures outbound messages as in-memory `SendRequest` objects on the success path instead of performing SMTP; combined with `NOT_SEND_EMAIL=true` this is why success proves enqueue, not SMTP hand-off.

### 2.5 Seeding the real data condition

Using the canonical helpers exactly as the tests do (`tests/utils.py: create_new_user`, `Alias.create_new_random`, `Contact.create`):

- `create_new_user(email=...)` creates a `User` whose **default mailbox email equals the user's email**. This behavior is implemented by `User.create` itself: it creates `mb = Mailbox.create(user_id=user.id, email=user.email, verified=True)` (`app/models.py:611`) and sets `user.default_mailbox_id = mb.id` (`app/models.py:613`). Consequently `get_mailbox_from_mail_from(user.email, alias)` matches that mailbox when — and only when — the alias belongs to that same user.
- `Alias.create_new_random(user)` creates one alias per user (the boot logs show the generated alias emails, e.g. `generate email word_list673@sl.local`).
- Two `Contact` rows are created on aliases owned by **different** users but with the **same** `reply_email`, via direct `Contact.create(...)`. This is accepted because there is no `UNIQUE` constraint on `reply_email` (proof in §6.1) and because `Contact.create` guards only that `website_email` is not itself a reverse-alias — it does **not** check `reply_email` uniqueness.

### 2.6 Observation classes: canonical, `[non-canonical]`, `[inferred]`

- **Canonical** observations drive the real inbound reply entry point `email_handler.handle_reply(envelope, msg, rcpt_to)` (`email_handler.py:966`); the routing claim is additionally proven by invoking the real hub `email_handler.handle(envelope, msg)` (`email_handler.py:1945`), which detects the reverse-alias and dispatches to `handle_reply()` (§7 is driven through `handle_reply`; §3–§5 use `handle_reply`, and the routing proof in §4.3 uses `handle()`).
- **`[non-canonical]`** observations — a direct `Contact.get_by(reply_email=...)`, a direct `is_reverse_alias(...)` predicate call, or the per-alias **spoof-check-disabled fallback** — are explicitly labeled `[non-canonical]` wherever they appear. They corroborate the underlying `.first()` mechanism or the selection/logging behavior but do not, by themselves, represent default canonical delivery.
- **`[inferred]`** statements (e.g., "a different query plan or physical layout could return the other row") are labeled as such and grounded in the official semantics cited in §4.1 and §9.
- **Scale.** The cross-event experiment drives the identical input **N=100** per run and repeats **each condition across 2 independent drive runs** against the **same** persisted rows (§7). Every seed and every drive run performed against the clean-slate database is disclosed in §7 — none is omitted.

### 2.7 Temporary harness scripts (thirteen), safety, and the reproducible-command convention

Thirteen throwaway scripts were used — twelve Python harnesses plus one `psql` SQL helper (`contact_schema.sql`). They live **outside** the repository at `/tmp/harness/` on the host (copied into the container's `/tmp/`), and all were removed afterward with per-file absence proof (§13). Their **full source is reproduced in §11.3** so results remain reproducible after deletion:

- `_bootstrap.py` — shared fail-fast bootstrap (documented below).
- `observe_reply.py` — canonical baseline + normalized-value provenance (two `handle_reply()` calls; handler-local `reply_email` captured by non-mutating `sys.settrace`) (§3–§5).
- `observe_wronguser.py` — CRITICAL default-control vs `[non-canonical]` fallback (§10).
- `observe_handle.py` — canonical routing through `handle()` (§4.3).
- `observe_dist.py` — same-unchanged-input distribution, seed-once/drive-many (§7).
- `observe_edge.py` — E501 / E502(no-contact) / E502(inactive-user) / `is_reverse_alias` (§8.1).
- `observe_norm.py` — inbound normalization / exact-equality lookup (§8.2).
- `observe_avail.py` — exact `available_sl_email` body, call sites, guard bypass (§6.2).
- `observe_race.py` — GENUINE concurrent check→commit race (two processes; fixed + generator leader/follower) proving a duplicate `reply_email` commits with no `UNIQUE` (§6.2.1).
- `observe_f07.py` — five-way E214 sender-authorization matrix, reachable E504, adversarial `is_reverse_alias()` and `replace_header_when_reply()` (§8.4–§8.8).
- `observe_sec.py` — F-04 security/privacy matrix: adversarial recipients (SQL metacharacter / CRLF / Unicode / overlong / null-byte) via the canonical `handle()`, plus cross-user E214 disclosure with explicit six-way separation (§8.9–§8.10).
- `observe_second.py` — the second lookup site `replace_header_when_reply()` (§8.3).
- `contact_schema.sql` — `psql` catalog helper for the §6.1(b) `reply_email`-has-no-`UNIQUE` runtime proof (read-only; no app bootstrap).

**Fail-fast safety (addresses the disposable-target concern).** Because exported env values override dotenv (§2.2), every harness **refuses to run** unless `DB_URI` is one of a small allowlist of disposable localhost test DSNs, **and** re-checks `current_database() == 'test'` after `create_app()`, **before** any schema or data write. It uses `CREATE EXTENSION IF NOT EXISTS` (never `DROP EXTENSION`). The shared fragment (reproduced verbatim; full per-harness copies in §11.3):

```python
# Shared corrected bootstrap fragment (documented inline in each harness).
import os
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
_DSN = os.environ.get("DB_URI", "")
if _DSN not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={_DSN!r} is not an allowlisted disposable test DB\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()

with engine.connect() as _c:
    _dbname = _c.execute("select current_database()").scalar()
if _dbname != "test":
    sys.stderr.write(f"REFUSING TO RUN: current_database()={_dbname!r} != 'test'\n")
    sys.exit(3)

with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")


def truncate_clean_slate():
    """Clean slate (deterministic IDs, only this run's rows). Safe: allowlist +
    current_database() guards already ran. Uses engine.begin() so the TRUNCATE
    commits and holds no lingering lock."""
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )
```

**Reproducible-command convention.** Each behavioral section below shows the **exact** command that produced its output. The following generic form is a **template only (not a literal reproducible command)** — the concrete, runnable commands (with real script names and arguments) appear inline in §3–§8 and §11:

```
# TEMPLATE (illustrative only — see each section for the exact command):
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_<name>.py [args] 2>&1'
```

---
### 2.8 Self-contained clean-room bootstrap (fresh-container replay, F-08)

Every value in this document was captured inside the canonical **warmed** container, which the image's `build.sh` had already set up. So that the document is a **self-contained runbook** — reproducible from the stated base image with **no hidden state** — this subsection enumerates and **verifies** the complete ordered bring-up. Five setup classes must be applied before the command sequences elsewhere reproduce their embedded outputs: **(1)** service startup, **(2)** the runtime env file, **(3)** DB role/schema/extensions/migrations, **(4)** restoration of three `git`-tracked key fixtures that `build.sh` clobbers, and **(5)** materialization of the temporary harnesses (deleted by the read-only cleanup, §13). Omitting **(4)** is what makes DKIM-signing reply-forward invocations fail with `dkim.KeyFormatError`; the other four cause connection/import/collection failures.

**Ordered bootstrap:**

```bash
# ---- 0. BASE IMAGE (per user setup) ----
#   image:  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0
#   /app is the SimpleLogin checkout at commit 2cd6ee77; venv at /app/venv.

# ---- 1. START SERVICES ----
service postgresql start
service redis-server start

# ---- 2. SOURCE THE RUNTIME ENV (CONFIG, DB_URI, EMAIL_DOMAIN, key paths) ----
cd /app && . /app/venv/bin/activate
set -a && . /tmp/sl_env.sh && set +a
#   DB_URI resolves to ...@localhost:5432/test, so NO socat 15432 bridge is needed (see §2.2).

# ---- 3. DB ROLE / SCHEMA / EXTENSIONS / MIGRATIONS ----
#   On a PRISTINE container only (avoids the pg_trgm migration-ordering bug): recreate the
#   'test' role+db and uuid-ossp, then reset the schema BEFORE migrating, and do NOT
#   pre-create pg_trgm (the migration creates it):
#     psql -c "drop schema public cascade; create schema public;"   # fresh DB only
alembic upgrade head          # -> 32f25cbf12f6 ; pg_trgm is created by the migrations
#   (The harness bootstrap also runs CREATE EXTENSION IF NOT EXISTS pg_trgm idempotently and
#    never DROPs it -- see §2.7 fail-fast safety.)

# ---- 4. RESTORE THE THREE TRACKED KEY FIXTURES (build.sh clobbers them) ----
#   WITHOUT this, every DKIM-signing reply-forward invocation raises dkim.KeyFormatError
#   (the 11/19-style failures). These files are git-tracked, so a checkout restores them:
git checkout -- local_data/dkim.key local_data/paddle.key.pub local_data/private-pgp.asc

# ---- 5. MATERIALIZE THE TEMPORARY HARNESSES ----
#   The observation scripts are deleted by the read-only cleanup (§13). Recreate each
#   /tmp/observe_<name>.py (and /tmp/contact_schema.sql) VERBATIM from its full source in §11.3.

# ---- 6. RUN ANY HARNESS (reproducible-command convention, §2.7) ----
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_<name>.py [args] 2>&1'
```

**Verification of steps 1-4 in the live canonical container (complete, unedited):**

```
### 1. services ###
/var/run/postgresql:5432 - accepting connections
redis-cli ping: PONG

### 2. effective env (from /tmp/sl_env.sh) ###
CONFIG=/app/tests/test.env
DB_URI=postgresql://test:test@localhost:5432/test
EMAIL_DOMAIN=sl.local

### 3. DB role / database / extensions + alembic head ###
current_user | current_database: test|test
extensions: pg_trgm plpgsql
alembic current: 32f25cbf12f6 (head)

### 4. tracked key fixtures (restored state) ###
git HEAD: 2cd6ee777f8c
1c86748b8e445f96...  local_data/dkim.key  (886 bytes, tracked)
9d5d5028703f3289...  local_data/paddle.key.pub  (799 bytes, tracked)
666b8d71c1fee6e8...  local_data/private-pgp.asc  (3498 bytes, tracked)
```

This confirms: PostgreSQL accepting on `5432` and Redis `PONG` (step 1); `CONFIG`/`DB_URI`/`EMAIL_DOMAIN` exported (step 2); role/db `test`/`test`, extensions `pg_trgm`+`plpgsql`, and Alembic at head **`32f25cbf12f6`** (step 3); and the three key fixtures present and git-tracked at commit `2cd6ee777f8c` (step 4).

**Verification that step 4 is load-bearing (the `KeyFormatError` failure → `git checkout` fix, reproduced live and non-destructively).** `app/config.py:186-189` reads `DKIM_PRIVATE_KEY` from `local_data/dkim.key`; `app/email_utils.py:490-495` feeds it to `dkim.sign()` on the reply-forward path. When `build.sh` overwrites the key with a non-PEM placeholder, that call raises. The cycle below clobbers the (git-tracked) key, shows the failure, restores it with the step-4 command, and shows the path working again — leaving the file **byte-identical** to its tracked state (`sha256 1c86748b…` before and after):

```
### STEP 0 — baseline: dkim.key sha256 + signing works ###
1c86748b8e445f96c4caaf1742a3d29fd628e5c08fc0dd9f0b45dd49a10326a6  local_data/dkim.key
SIGN OK

### STEP 1 — clobber dkim.key (simulating build.sh overwrite with a non-PEM placeholder) ###
6bef63d015cb8e6e4878cd925a08680882958a6a73c82314cd4f2d02b9e87dd1  local_data/dkim.key

### STEP 2 — a DKIM-signing reply-forward path now FAILS (11/19-style) ###
dkim.crypto.UnparsableKeyError: Private key not found
    raise KeyFormatError(str(e))
dkim.KeyFormatError: Private key not found

### STEP 3 — documented restore: git checkout -- local_data/dkim.key ###
1c86748b8e445f96c4caaf1742a3d29fd628e5c08fc0dd9f0b45dd49a10326a6  local_data/dkim.key

### STEP 4 — signing path works again (restored) ###
SIGN OK
```

The middle block is exactly the `dkim.KeyFormatError: Private key not found` that QA observed for 11 of 19 signing invocations before restoration; `git checkout` returns the key to `sha256 1c86748b…` and signing succeeds again.

**Cleanup after replay.** The harnesses of step 5 and any scratch are removed per §13 (which carries the per-file absence proof); the git-tracked keys restored in step 4 are already at their committed state (`git status` clean), and the container itself is disposable. With steps 1-5 applied, the exact command sequences throughout this document reproduce their embedded outputs from the stated base image with no hidden state.

---
## 3. Reply-address derivation

The reply address is the inbound SMTP recipient, taken verbatim, then domain-gated and normalized. An annotated excerpt of the canonical source (commit `2cd6ee777f8c`, re-confirmed at runtime) — intervening unchanged lines are elided and the original line numbers are preserved:

```
email_handler.py:966   def handle_reply(envelope, msg: Message, rcpt_to: str) -> (bool, str):
email_handler.py:972       reply_email = rcpt_to
email_handler.py:974       reply_domain = get_email_domain_part(reply_email)
email_handler.py:977       if not reply_email.endswith(EMAIL_DOMAIN):
email_handler.py:978           sl_domain: SLDomain = SLDomain.get_by(domain=reply_domain)
email_handler.py:980               LOG.w(f"Reply email {reply_email} has wrong domain")
email_handler.py:981               return False, status.E501
email_handler.py:984       reply_email = normalize_reply_email(reply_email)
email_handler.py:986       contact = Contact.get_by(reply_email=reply_email)
```

- **Extraction:** `reply_email = rcpt_to` (`email_handler.py:972`). The reply address is *literally* the envelope recipient — no transformation at this step.
- **Domain gate:** `if not reply_email.endswith(EMAIL_DOMAIN)` (`email_handler.py:977`). If the address does not end with `EMAIL_DOMAIN` (`sl.local`), the handler falls back to an `SLDomain` lookup on the domain part (`email_handler.py:978`) and returns `status.E501` if that also fails (`email_handler.py:981`). The observed E501 is in §8.1.
- **Normalization:** `reply_email = normalize_reply_email(reply_email)` (`email_handler.py:984`). `normalize_reply_email()` (`app/email_validation.py:25`) first routes non-ASCII input through `convert_to_id()`, then replaces any character **not** in `_ALLOWED_CHARS` (`app/email_validation.py:9`) with `_`. This is a **many-to-one** mapping applied to the **inbound** string only (its precise semantics, and what it does and does not imply for multi-row matches, are established with runtime evidence in §8.2).

The observed extracted value (the raw `rcpt_to` at `:972`) and the observed **normalized** value are reported in §4.2 from `handle_reply()`'s **own frame local**, captured by a non-mutating `sys.settrace` hook: the trace prints `reply_email` at `:974` (raw, post-`:972`) and again at `:986` (the value **after** the `:984` normalize, i.e. the exact string handed to `Contact.get_by`). A direct `normalize_reply_email(...)` call is shown in §4.2 only as an explicitly `[non-canonical]` cross-check. §4.2 CASE 2 exhibits a raw≠normalized input (`dup#norm#core@sl.local` → `dup_norm_core@sl.local`) so the trace visibly changes at `:986`, proving the reported normalized value is genuinely the handler's post-`:984` local rather than the raw input or a helper echo.

---
## 4. Contact resolution & `.first()` semantics

### 4.1 The lookup and its underlying `.first()`

The normalized reply address resolves a **single** `Contact`:

```
email_handler.py:986   contact = Contact.get_by(reply_email=reply_email)
email_handler.py:988       LOG.w(f"No contact with {reply_email} as reverse alias")
email_handler.py:989       return False, status.E502            # when no contact is found
email_handler.py:990   if not contact.user.is_active():
email_handler.py:991       LOG.w(f"User {contact.user} has been soft deleted")
email_handler.py:992       return False, status.E502
```

`Contact.get_by(...)` is the inherited `ModelMixin.get_by()` helper. Its **entire body** is a `.first()` with **no `ORDER BY`** (verbatim via `inspect.getsource` in §6.2):

```
app/models.py:82      @classmethod
app/models.py:83      def get_by(cls, **kw):
app/models.py:84          return Session.query(cls).filter_by(**kw).first()
```

**Framework semantics (official, for the pinned SQLAlchemy 1.3.24).** `Query.first()` applies a `LIMIT 1` and returns the first row the database yields ([SQLAlchemy 1.3 Query API — `Query.first()`](https://docs.sqlalchemy.org/en/13/orm/query.html)). When no `ORDER BY` is present and more than one row matches, which row is "first" is **not deterministic**: the SQLAlchemy 1.3 FAQ states that "*A relational database can return rows in any arbitrary order, when an explicit ordering is not set*" and that "*any query that limits rows using LIMIT or OFFSET should always specify an ORDER BY. Otherwise, it is not deterministic which rows will actually be returned*," recommending an `ORDER BY` on a unique column (the primary key) as the remedy ([SQLAlchemy 1.3 FAQ — "Why is ORDER BY required with LIMIT"](https://docs.sqlalchemy.org/en/13/faq/ormconfiguration.html#why-is-order-by-required-with-limit-especially-with-subqueryload)). SimpleLogin's `get_by()` specifies no such order, so on a multi-row `reply_email` match the resolved `Contact` is unordered at the application level (the PostgreSQL-level determinant is analyzed with `EXPLAIN`/`ctid` evidence in §7.4).

### 4.2 Observed resolved contact — the canonical happy-path call (complete, unedited output)

The harness `observe_reply.py` seeds the real duplicate condition (two contacts sharing one `reply_email` on aliases owned by different users), proves the duplicate persisted, then drives the **canonical** entry point `email_handler.handle_reply(envelope, msg, rcpt_to)` **twice** with `mail_from = user_A`'s mailbox:

- **CASE 1 (baseline, raw == normalized):** `rcpt_to='dup-reply-core@sl.local'` — the correct-owner happy path against the duplicate pair (owner of the first-inserted contact's alias).
- **CASE 2 (normalized-value *provenance*, raw ≠ normalized):** `rcpt_to='dup#norm#core@sl.local'`. Because `#` is **not** in `_ALLOWED_CHARS` (`app/email_validation.py:9`), the `:984` normalize step maps it to `_`, so the value the handler actually looks up is `'dup_norm_core@sl.local'`. The single seeded contact carries that **normalized** value.

**Normalized-value provenance (this is the canonical evidence, not a helper call).** The reported normalized `reply_email` is captured from `handle_reply()`'s **own frame local** by a **non-mutating** `sys.settrace` line hook that reads `frame.f_locals['reply_email']` — recording the value when it changes **and** always snapshotting it at the `Contact.get_by` line (`email_handler.py:986`), where the local holds the post-`:984` normalized value. The tracer only *observes* frames; it mutates no product code (it is installed with `sys.settrace(...)` immediately before the call and removed with `sys.settrace(None)` immediately after). A direct `normalize_reply_email(...)` call is printed **only** as an explicitly `[non-canonical]` cross-check; it is not the evidence. In CASE 2 the trace **visibly changes** from `'dup#norm#core@sl.local'` (`:974`, raw) to `'dup_norm_core@sl.local'` (`:986`, normalized), proving the reported value is genuinely the handler's post-normalize local. Direct ORM/predicate corroboration remains explicitly `[non-canonical]`.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_reply.py 2>&1'
```

**Output (complete, unedited; run 1 of 2 — run-2 stability is machine-verified immediately below, not merely asserted):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 01:47:04,085 - SL - DEBUG - 14849 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 01:47:08,586 - SL - INFO - 14849 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 01:47:08,589 - SL - INFO - 14849 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 01:47:08,590 - SL - INFO - 14849 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 01:47:08,592 - SL - DEBUG - 14849 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 01:47:08,878 - SL - INFO - 14849 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 01:47:09,143 - SL - INFO - 14849 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 01:47:09,157 - SL - DEBUG - 14849 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_list673@sl.local
2026-07-14 01:47:09,164 - SL - INFO - 14849 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 01:47:09,174 - SL - DEBUG - 14849 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_test952@sl.local
2026-07-14 01:47:09,182 - SL - INFO - 14849 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 01:47:09,190 - SL - DEBUG - 14849 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_list212@sl.local
2026-07-14 01:47:09,198 - SL - INFO - 14849 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== SEED (insertion order A-then-B) ===
EMAIL_DOMAIN='sl.local'  NOT_SEND_EMAIL=True
CASE1 shared_reply='dup-reply-core@sl.local'
CASE2 raw_reply='dup#norm#core@sl.local'  norm_reply(stored)='dup_norm_core@sl.local'
user_a.id=1 email='usera@mailbox.test' alias_a.id=3 alias_a.email='word_list673@sl.local'
user_b.id=2 email='userb@mailbox.test' alias_b.id=4 alias_b.email='test_test952@sl.local'
alias_a2.id=5 alias_a2.email='list_list212@sl.local' (also owned by user_a)
contact_a.id=1 (alias_a,user_a)  contact_b.id=2 (alias_b,user_b)  contact_n.id=3 (alias_a2,user_a)
=== PROVE CASE1 DUPLICATE PERSISTED (no UNIQUE constraint blocked it) ===
rows sharing reply_email='dup-reply-core@sl.local': count=2
  Contact id=1 alias_id=3 user_id=1 website_email='a@nowhere.net'
  Contact id=2 alias_id=4 user_id=2 website_email='b@nowhere.net'
=== raw reply-address derivation inputs (email_handler.py:972,977) ===
CASE1 rcpt_to.endswith(EMAIL_DOMAIN) = True
CASE2 rcpt_to.endswith(EMAIL_DOMAIN) = True

=== [CASE1 baseline raw==normalized] CANONICAL CALL: email_handler.handle_reply(envelope, msg, rcpt_to) ===
rcpt_to (raw)                 = 'dup-reply-core@sl.local'
envelope.mail_from            = 'usera@mailbox.test'
Message-ID                    = '<obs-core-0@sl.local>'
[non-canonical] normalize_reply_email('dup-reply-core@sl.local') = 'dup-reply-core@sl.local'  (direct helper cross-check; NOT the canonical evidence)
2026-07-14 01:47:09,223 - SL - INFO - 14849 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 01:47:09,232 - SL - DEBUG - 14849 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 a@nowhere.net 3>, <User 1 Test User usera@mailbox.test>, <Mailbox 1 usera@mailbox.test>
2026-07-14 01:47:09,244 - SL - DEBUG - 14849 - "/app/email_handler.py:1171" - handle_reply() -  - From header is word_list673@sl.local
2026-07-14 01:47:09,244 - SL - DEBUG - 14849 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the To header. Old value word_list673@sl.local
2026-07-14 01:47:09,245 - SL - DEBUG - 14849 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-14 01:47:09,248 - SL - DEBUG - 14849 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178399362924.14849.5590649886963581070.1@sl.local>
2026-07-14 01:47:09,255 - SL - WARNING - 14849 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-14 01:47:09,260 - SL - DEBUG - 14849 - "/app/email_handler.py:1212" - handle_reply() -  - send email from word_list673@sl.local to a@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-14 01:47:09,265 - SL - DEBUG - 14849 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply subject', from 'word_list673@sl.local' to 'None'
RESULT delivered=True code='250 Message accepted for delivery'
code==status.E200? True  ==E214? False  ==E502? False
--- [CASE1 baseline raw==normalized] CANONICAL handler-local reply_email (non-mutating sys.settrace) ---
  email_handler.py:972  reply_email = '<unset>'
  email_handler.py:974  reply_email = 'dup-reply-core@sl.local'
  email_handler.py:986  reply_email = 'dup-reply-core@sl.local'  <- post-:984 normalized value used by Contact.get_by(:986)
CANONICAL normalized reply_email (handler local at :986) = 'dup-reply-core@sl.local'
PERSISTED EmailLog (message_id='<obs-core-0@sl.local>'): id=1 contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  correlate: raw rcpt_to='dup-reply-core@sl.local' -> handler-local normalized='dup-reply-core@sl.local' -> resolved contact_id=1 -> forwarding user_id=1 (user_a.id=1, user_b.id=2)
stored SendRequest count = 1
  SendRequest envelope_from='sl.lmysyibrfqqdemzygmztan25.srzh6sgrqbbqe@sl.local' envelope_to='a@nowhere.net' msg[From]='word_list673@sl.local'
NEW SentAlert rows during call: 0
handle_reply() wall-clock: 47.9 ms

=== [CASE2 provenance raw!=normalized] CANONICAL CALL: email_handler.handle_reply(envelope, msg, rcpt_to) ===
rcpt_to (raw)                 = 'dup#norm#core@sl.local'
envelope.mail_from            = 'usera@mailbox.test'
Message-ID                    = '<obs-core-norm-0@sl.local>'
[non-canonical] normalize_reply_email('dup#norm#core@sl.local') = 'dup_norm_core@sl.local'  (direct helper cross-check; NOT the canonical evidence)
2026-07-14 01:47:09,280 - SL - INFO - 14849 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 01:47:09,284 - SL - DEBUG - 14849 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 2> for <Contact 3 n@nowhere.net 5>, <User 1 Test User usera@mailbox.test>, <Mailbox 1 usera@mailbox.test>
2026-07-14 01:47:09,293 - SL - DEBUG - 14849 - "/app/email_handler.py:1171" - handle_reply() -  - From header is list_list212@sl.local
2026-07-14 01:47:09,294 - SL - DEBUG - 14849 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the To header. Old value list_list212@sl.local
2026-07-14 01:47:09,294 - SL - DEBUG - 14849 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-14 01:47:09,296 - SL - DEBUG - 14849 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178399362929.14849.8126660391972311832.2@sl.local>
2026-07-14 01:47:09,301 - SL - WARNING - 14849 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-14 01:47:09,306 - SL - DEBUG - 14849 - "/app/email_handler.py:1212" - handle_reply() -  - send email from list_list212@sl.local to n@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-14 01:47:09,311 - SL - DEBUG - 14849 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply subject', from 'list_list212@sl.local' to 'None'
RESULT delivered=True code='250 Message accepted for delivery'
code==status.E200? True  ==E214? False  ==E502? False
--- [CASE2 provenance raw!=normalized] CANONICAL handler-local reply_email (non-mutating sys.settrace) ---
  email_handler.py:972  reply_email = '<unset>'
  email_handler.py:974  reply_email = 'dup#norm#core@sl.local'
  email_handler.py:986  reply_email = 'dup_norm_core@sl.local'  <- post-:984 normalized value used by Contact.get_by(:986)
CANONICAL normalized reply_email (handler local at :986) = 'dup_norm_core@sl.local'
PERSISTED EmailLog (message_id='<obs-core-norm-0@sl.local>'): id=2 contact_id=3 alias_id=5 user_id=1 mailbox_id=1 is_reply=True
  correlate: raw rcpt_to='dup#norm#core@sl.local' -> handler-local normalized='dup_norm_core@sl.local' -> resolved contact_id=3 -> forwarding user_id=1 (user_a.id=1, user_b.id=2)
stored SendRequest count = 1
  SendRequest envelope_from='sl.lmysyibsfqqdemzygmztan25.adfnh67r3uvai@sl.local' envelope_to='n@nowhere.net' msg[From]='list_list212@sl.local'
NEW SentAlert rows during call: 0
handle_reply() wall-clock: 35.8 ms

NOTE: NOT_SEND_EMAIL=True => mail_sender.send() logs and returns True WITHOUT calling _send_to_smtp; no external SMTP delivery is confirmed (app/mail_sender.py:130-136).
DONE
```

**Run-2 stability (machine-verified, not asserted).** The identical harness was executed in a **second, fresh OS process** (`observe_reply.run2.out`). The two runs were compared after masking only the fields that are *legitimately* per-run-varying — logger timestamps + PID, the randomly-generated alias local-parts, the random reverse-alias token in `envelope_from`, the generated `sl_message_id`, and the `handle_reply()` wall-clock ms:

```
$ maskdoc() { sed -E \
    "s/^20[0-9]{2}-[0-9]{2}-[0-9]{2} [0-9:,]+ - SL - [A-Z]+ - [0-9]+ - /<TS> - SL - LEVEL - <PID> - /; \
     s/[a-z]+_[a-z]+[0-9]+@sl\.local/<RANDOM_ALIAS>@sl.local/g; \
     s/sl\.[a-z0-9]+\.[a-z0-9]+@sl\.local/<RANDOM_REVERSE_ALIAS>@sl.local/g; \
     s/<1783[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+@sl\.local>/<SL_MESSAGE_ID>/g; \
     s/wall-clock: [0-9.]+ ms/wall-clock: <MS> ms/g" "$1"; }
$ diff <(maskdoc observe_reply.run1.out) <(maskdoc observe_reply.run2.out); echo "exit=$?"
exit=0
$ sha256sum <(maskdoc observe_reply.run1.out) <(maskdoc observe_reply.run2.out)
7d42a9a8bc46fb7e16aa5228c66c719d5cd5e1c8bae68efb65108da928e0059d  /dev/fd/63
7d42a9a8bc46fb7e16aa5228c66c719d5cd5e1c8bae68efb65108da928e0059d  /dev/fd/62
```

`diff` produced **no output (exit 0)** and both masked runs share one SHA256 — so every *semantic* value (the handler-local `reply_email` trace, the resolved `contact_id`/`alias_id`/`user_id`, the status code, and the persisted `EmailLog`) is **byte-identical across two fresh processes**; only the enumerated random/timing fields differ.

**Reading the output (each value grounded):**

- **Duplicate persisted (no `UNIQUE` blocked it):** `rows sharing reply_email='dup-reply-core@sl.local': count=2` — `Contact id=1` (`alias_id=3`, `user_id=1`, `a@nowhere.net`) and `Contact id=2` (`alias_id=4`, `user_id=2`, `b@nowhere.net`). This is the real data condition the whole investigation rests on, and it is accepted by the schema (§6.1).
- **Reply-address derivation — the raw input (`email_handler.py:972,977`):** `rcpt_to (raw)` is the address SimpleLogin assigns `reply_email` at `:972`; the `:977` domain gate observes `rcpt_to.endswith(EMAIL_DOMAIN) = True` for both cases (`dup-reply-core@sl.local` and `dup#norm#core@sl.local`).
- **CANONICAL normalized value — captured from `handle_reply()`'s own frame local (`email_handler.py:984` → observed at `:986`):** this is the F-critical value and it comes from the **non-mutating `sys.settrace`** trace of the live handler frame, **not** from a direct helper call:
  - **CASE 1 (raw == normalized):** the handler local is `'dup-reply-core@sl.local'` at `:974` and still `'dup-reply-core@sl.local'` at `:986` (every character is already in `_ALLOWED_CHARS`, so `:984` is a no-op here). `CANONICAL normalized reply_email (handler local at :986) = 'dup-reply-core@sl.local'`.
  - **CASE 2 (raw ≠ normalized — provenance proof):** the handler local is `'dup#norm#core@sl.local'` at `:974` and **changes to `'dup_norm_core@sl.local'` at `:986`** (the `:984` normalize mapped both `#` to `_`). `CANONICAL normalized reply_email (handler local at :986) = 'dup_norm_core@sl.local'`. The visible change at the witness line proves the reported value is the handler's **post-`:984`** local — it is impossible for this to be the raw `rcpt_to` or a coincidental helper echo.
- **`[non-canonical]` cross-checks (bypass the entry point, explicitly labeled, NOT the evidence):** the direct helper `normalize_reply_email('dup-reply-core@sl.local') = 'dup-reply-core@sl.local'` and `normalize_reply_email('dup#norm#core@sl.local') = 'dup_norm_core@sl.local'` merely *agree with* the canonical handler-local values above; they are printed for corroboration only. (The `is_reverse_alias(...)` predicate and a direct `Contact.get_by(...)` are likewise available as `[non-canonical]` corroboration; the canonical routing/resolution proofs are §4.3 and the trace above.)
- **Canonical resolution:** the real `handle_reply()` call resolves the correct contact and logs at `email_handler.py:1051` — CASE 1: `Create <EmailLog 1> for <Contact 1 a@nowhere.net 3>, <User 1 ... usera@mailbox.test>, <Mailbox 1 ...>`; CASE 2: `Create <EmailLog 2> for <Contact 3 n@nowhere.net 5>, <User 1 ...>, <Mailbox 1 ...>`.
- **Persisted `EmailLog`, correlated by exact Message-ID:** CASE 1 (`<obs-core-0@sl.local>`) → `id=1 contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True`; CASE 2 (`<obs-core-norm-0@sl.local>`) → `id=2 contact_id=3 alias_id=5 user_id=1 mailbox_id=1 is_reply=True`. Both forward to `user_id=1` (the correct owner), and the full chain is printed: `raw rcpt_to -> handler-local normalized -> resolved contact_id -> forwarding user_id`.
- **Application acceptance vs. delivery:** both calls return `delivered=True code='250 Message accepted for delivery'` (`==status.E200? True`). One `SendRequest` was stored per call (`envelope_to='a@nowhere.net'` / `'n@nowhere.net'` = the resolved `contact.website_email`). The explicit `NOTE` records that with `NOT_SEND_EMAIL=True`, `mail_sender.send()` logs and returns `True` **without** calling `_send_to_smtp` (`app/mail_sender.py:130-136`) — so no external SMTP delivery is confirmed. `NEW SentAlert rows during call: 0`.

**Two diagnostics explained (so the log is not misread):**

- **`missing date header, add one` (`email_handler.py:1206`, `WARNING`).** The synthetic reply message carries no `Date:` header, so `handle_reply()` adds one before sending. It is an expected normalization for a hand-constructed message, not an error.
- **`send email ... to 'None'` (`app/mail_sender.py:131`) while `envelope_to='a@nowhere.net'`.** These refer to *different* things. The mail-sender log prints the **message header** `To:` (`msg[headers.TO]`), which is `None` here because `replace_header_when_reply()` deleted the reverse-alias `To` header (`email_handler.py:383` — `delete the To header`) since the original `To` was the reverse-alias itself. The **SMTP envelope recipient** is separate and is the real contact address: `SendRequest.envelope_to='a@nowhere.net'` (`= contact.website_email`). In other words, the *message header* being `None` and the *envelope recipient* being the contact are simultaneously correct and non-contradictory.

### 4.3 Canonical routing proof — `email_handler.handle()` dispatches to `handle_reply()`

To prove the routing claim rather than merely corroborate it from source, `observe_handle.py` invokes the **real routing hub** `email_handler.handle(envelope, msg)` (not `handle_reply` directly) with a reverse-alias recipient, and observes the hub detect the reverse-alias and dispatch into the reply phase.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_handle.py 2>&1'
```

**Output (complete, unedited; run 1 of 2 — run-2 stability is machine-verified immediately below, not merely asserted):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 01:53:18,498 - SL - DEBUG - 14908 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 01:53:22,813 - SL - INFO - 14908 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 01:53:22,817 - SL - INFO - 14908 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 01:53:22,818 - SL - INFO - 14908 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 01:53:22,819 - SL - DEBUG - 14908 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 01:53:23,105 - SL - INFO - 14908 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 01:53:23,122 - SL - DEBUG - 14908 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_list873@sl.local
2026-07-14 01:53:23,129 - SL - INFO - 14908 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== CANONICAL ROUTING: email_handler.handle(envelope, msg) (NOT handle_reply directly) ===
reply_email(reverse-alias)='route-check@sl.local'  mail_from='usera@mailbox.test'  Message-ID='<route-0@sl.local>'
2026-07-14 01:53:23,137 - SL - DEBUG - 14908 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-14 01:53:23,139 - SL - DEBUG - 14908 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:usera@mailbox.test, rcpt_tos:['route-check@sl.local'], header_from:a@nowhere.net, header_to:route-check@sl.local, cc:None, reply-to:None, message_id:<route-0@sl.local>, client_ip:None, headers:[('From', 'a@nowhere.net'), ('To', 'route-check@sl.local'), ('Message-ID', '<route-0@sl.local>'), ('Subject', 'reply via handle()'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 01:53:23,143 - SL - DEBUG - 14908 - "/app/email_handler.py:2196" - handle() -  - Reply phase usera@mailbox.test(a@nowhere.net) -> route-check@sl.local
2026-07-14 01:53:23,145 - SL - INFO - 14908 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 01:53:23,151 - SL - DEBUG - 14908 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 a@nowhere.net 2>, <User 1 Test User usera@mailbox.test>, <Mailbox 1 usera@mailbox.test>
2026-07-14 01:53:23,158 - SL - DEBUG - 14908 - "/app/email_handler.py:1171" - handle_reply() -  - From header is word_list873@sl.local
2026-07-14 01:53:23,159 - SL - DEBUG - 14908 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: route-check@sl.local, new: A <a@nowhere.net>
2026-07-14 01:53:23,159 - SL - DEBUG - 14908 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-14 01:53:23,162 - SL - DEBUG - 14908 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178399400316.14908.11540983407821836725.1@sl.local>
2026-07-14 01:53:23,168 - SL - DEBUG - 14908 - "/app/email_handler.py:1212" - handle_reply() -  - send email from word_list873@sl.local to a@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-14 01:53:23,172 - SL - DEBUG - 14908 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply via handle()', from 'word_list873@sl.local' to 'A <a@nowhere.net>'
handle() returned SMTP status = '250 Message accepted for delivery'  (==E200? True)
handle() routed to handle_reply -> EmailLog id=1 contact_id=1 alias_id=2 user_id=1 is_reply=True
DONE
```

**Run-2 stability (machine-verified, not asserted).** The identical harness was executed in a **second, fresh OS process** (`observe_handle.run2.out`) and compared after masking only the legitimately per-run-varying fields (logger timestamps + PID, the randomly-generated alias local-part, and the generated `sl_message_id`):

```
$ maskh() { sed -E \
    "s/^20[0-9]{2}-[0-9]{2}-[0-9]{2} [0-9:,]+ - SL - [A-Z]+ - [0-9]+ - /<TS> - SL - LEVEL - <PID> - /; \
     s/[a-z]+_[a-z]+[0-9]+@sl\.local/<RANDOM_ALIAS>@sl.local/g; \
     s/<1783[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+@sl\.local>/<SL_MESSAGE_ID>/g" "$1"; }
$ diff <(maskh observe_handle.run1.out) <(maskh observe_handle.run2.out); echo "exit=$?"
exit=0
$ sha256sum <(maskh observe_handle.run1.out) <(maskh observe_handle.run2.out)
29fcbe90b752b21541b0ef2ed91fe1e77129af47cb8085f5a72c2d4096db540d  /dev/fd/63
29fcbe90b752b21541b0ef2ed91fe1e77129af47cb8085f5a72c2d4096db540d  /dev/fd/62
```

`diff` produced **no output (exit 0)** and both masked runs share one SHA256 — so the routing decision, the `Reply phase` dispatch, the SMTP status, and the persisted `EmailLog` are **byte-identical across two fresh processes**.

- The hub logs `==>> Handle mail_from:usera@mailbox.test, rcpt_tos:['route-check@sl.local'] ...` (`email_handler.py:1980`), then `Reply phase usera@mailbox.test(a@nowhere.net) -> route-check@sl.local` (`email_handler.py:2196`) — this is where `handle()` classifies the recipient as a reverse-alias and enters the reply phase — and dispatches to `handle_reply()`.
- Result: `handle() returned SMTP status = '250 Message accepted for delivery'` (`==E200? True`), and `handle_reply` created `EmailLog id=1 contact_id=1 alias_id=2 user_id=1 is_reply=True`. Routing is therefore **exercised**, not just inferred.

---
## 5. Forwarding-destination selection

The forwarding destination (alias, owning user, and sender mailbox) is derived **entirely from the resolved contact**. An annotated excerpt of the canonical source (commit `2cd6ee777f8c`, re-confirmed at runtime) — the E503 sanity branch is compressed and the original line numbers are preserved:

```
email_handler.py:994    alias = contact.alias
email_handler.py:1000       ... (E503 sanity: alias/domain consistency)
email_handler.py:1004   user = alias.user
email_handler.py:1007   if not user.can_send_or_receive():        # -> E504 when the user cannot send/receive
email_handler.py:1019   mailbox = get_mailbox_from_mail_from(mail_from, alias)
email_handler.py:1021       if alias.disable_email_spoofing_check:  # [non-canonical] fallback path
email_handler.py:1032       else: handle_unknown_mailbox(...)       # default control
email_handler.py:1034           return False, status.E214
email_handler.py:1042   email_log = EmailLog.create(
email_handler.py:1043       contact_id=contact.id, alias_id=contact.alias_id, is_reply=True,
email_handler.py:1046       user_id=contact.user_id, mailbox_id=mailbox.id, ... )
```

- **Alias** — `alias = contact.alias` (`email_handler.py:994`): the alias is whatever the resolved contact points at, via the `Contact.alias_id` foreign key.
- **User** — `user = alias.user` (`email_handler.py:1004`): the owning user is derived transitively from that alias. The persisted `EmailLog.user_id` is taken directly from `contact.user_id` (`email_handler.py:1046`), so the log records the **resolved contact's** owner regardless of who actually sent the reply.
- **Mailbox** — `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:1019`): the sender mailbox is looked up by matching the SMTP `mail_from` against mailboxes **authorized for that alias**. If the alias belongs to a *different* user than the sender's mailbox, this returns `None`, and the spoof-check branch (`email_handler.py:1019-1034`) decides the outcome:
  - **Default (`disable_email_spoofing_check = False`):** `handle_unknown_mailbox(...)` is called and the handler returns `status.E214` (`email_handler.py:1032-1034`) — the reply is rejected **before** `EmailLog.create` (§7 shows this as the default-control result for the wrong-user data condition).
  - **`[non-canonical]` fallback (`disable_email_spoofing_check = True`):** the check is skipped (log `ignore unknown sender ...` at `email_handler.py:1023`), the mailbox falls back to `alias.mailbox`, and the reply proceeds to `EmailLog.create` and a stored send request — under the resolved (possibly wrong) user.

**Observed forwarding destination (canonical happy path — CASE 1).** The values are in the §4.2 output block: `alias_id=3` (the run-1 random alias `word_list673@sl.local`), `user_id=1`, `mailbox_id=1` (`usera@mailbox.test`), and the rewritten `From` header `From header is word_list673@sl.local` (`email_handler.py:1171`). The `From` is rewritten to the alias so the recipient sees the alias, not the mailbox — matching the SimpleLogin reverse-alias contract (§9). The wrong-user variants of this selection (E214 default vs. `[non-canonical]` fallback forward) are shown with complete output in §7 and §10.

---
## 6. Uniqueness / race / timing root cause

The wrong-user outcome is not one bug but a chain: a permissive schema (no `UNIQUE`), a best-effort non-atomic generation-time guard (TOCTOU) that non-generator writes bypass, and an unordered `.first()` lookup that propagates the resolved row into the alias, user, mailbox, and log.

### 6.1 `reply_email` has **no** `UNIQUE` constraint (repository-wide search + runtime catalog proof)

**(a) Repository-wide search across migrations and the model.** Every `reply_email` reference in migrations and the only `Contact` unique constraint in the model:

**Command:**

```
docker exec sl_app bash -lc '
cd /app
echo "=== (1) All migration references to reply_email (text files only) ==="
grep -rn --include="*.py" "reply_email" migrations/versions/
echo ""
echo "=== (2) Any UNIQUE/unique constraint mentioning reply_email across migrations + models (case-insensitive) ==="
grep -rni --include="*.py" "reply_email" migrations/ app/models.py | grep -i "unique" || echo "(no line pairs reply_email with unique=True / UniqueConstraint)"
echo ""
echo "=== (3) The Contact reply_email column + the only Contact UniqueConstraint in the model ==="
grep -n "reply_email = sa.Column" app/models.py
grep -n "UniqueConstraint" app/models.py | grep -i "contact\|uq_contact"
'
```

**Output (complete, unedited; stable across run 1 and run 2):**

```
=== (1) All migration references to reply_email (text files only) ===
migrations/versions/2021_071310_78403c7b8089_.py:22:    op.create_index(op.f('ix_contact_reply_email'), 'contact', ['reply_email'], unique=False)
migrations/versions/2021_071310_78403c7b8089_.py:28:    op.drop_index(op.f('ix_contact_reply_email'), table_name='contact')
migrations/versions/5fa68bafae72_.py:28:    sa.Column('reply_email', sa.String(length=128), nullable=False),

=== (2) Any UNIQUE/unique constraint mentioning reply_email across migrations + models (case-insensitive) ===
migrations/versions/2021_071310_78403c7b8089_.py:22:    op.create_index(op.f('ix_contact_reply_email'), 'contact', ['reply_email'], unique=False)

=== (3) The Contact reply_email column + the only Contact UniqueConstraint in the model ===
1899:    reply_email = sa.Column(sa.String(512), nullable=False, index=True)
1875:        sa.UniqueConstraint("alias_id", "website_email", name="uq_contact"),
```

- The only index ever created on `reply_email` is `ix_contact_reply_email` with **`unique=False`** (`migrations/versions/2021_071310_78403c7b8089_.py:22`); the same migration drops it on downgrade (`:28`); the table-creating migration declares the column `nullable=False` with **no** unique flag (`migrations/versions/5fa68bafae72_.py:28`).
- The case-insensitive `reply_email`×`unique` cross-search returns **only** that `unique=False` index line — i.e., **no** `UniqueConstraint`/`unique=True` mentioning `reply_email` exists anywhere in migrations or the model.
- The model column is `reply_email = sa.Column(sa.String(512), nullable=False, index=True)` (`app/models.py:1899`) — indexed, not unique. The **only** `Contact` unique constraint is `sa.UniqueConstraint("alias_id", "website_email", name="uq_contact")` (`app/models.py:1875`).

**(b) Runtime catalog proof (`pg_indexes` + `pg_constraint`).** The live database confirms the same fact, run fail-fast with `-v ON_ERROR_STOP=1` (the full source of the `/tmp/contact_schema.sql` helper is reproduced in §11.3):

**Command:**

```
docker exec sl_app bash -lc 'su postgres -c "psql -d test -v ON_ERROR_STOP=1 -f /tmp/contact_schema.sql"; echo "psql_exit=$?"'
```

**Output (complete, unedited; stable across run 1 and run 2):**

```
Pager usage is off.
=== indexes on contact (pg_indexes) ===
       indexname        |                                    indexdef
------------------------+---------------------------------------------------------------------------------
 ix_contact_reply_email | CREATE INDEX ix_contact_reply_email ON public.contact USING btree (reply_email)
(1 row)

=== constraints on contact (pg_constraint) ===
        conname        | contype |                              def
-----------------------+---------+---------------------------------------------------------------
 contact_alias_id_fkey | f       | FOREIGN KEY (alias_id) REFERENCES alias(id) ON DELETE CASCADE
 contact_user_id_fkey  | f       | FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
 forward_email_pkey    | p       | PRIMARY KEY (id)
 uq_contact            | u       | UNIQUE (alias_id, website_email)
(4 rows)

=== is there ANY unique index/constraint covering reply_email? ===
 unique_on_reply_email
-----------------------
                     0
(1 row)

psql_exit=0
```

- The only index on `contact` touching `reply_email` is the **non-unique** `ix_contact_reply_email` (`USING btree (reply_email)` — no `UNIQUE`).
- The `contact` constraints are two foreign keys, the primary key `forward_email_pkey`, and exactly one unique constraint `uq_contact = UNIQUE (alias_id, website_email)`.
- The explicit count query answers the question directly: `unique_on_reply_email = 0`. The script exits `psql_exit=0` (no `ON_ERROR_STOP` abort).

### 6.2 The generation-time guard is a best-effort TOCTOU check — and a direct write bypasses it

`available_sl_email()` is the only application-level uniqueness gate for a new reverse-alias value. Its **exact body** (via `inspect.getsource`, so no check is hidden behind an ellipsis), the exact body of `ModelMixin.get_by()`, the exact **two-clause** body of `is_reverse_alias()`, and a direct-write bypass demonstration:

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_avail.py 2>&1'
```

**Output (complete, unedited; stable across run 1 and run 2):**

```
########## observe_avail.py RUN 1 ##########
=== EXACT body of available_sl_email() [app/models.py] (inspect.getsource) ===
  1425: def available_sl_email(email: str) -> bool:
  1426:     if (
  1427:         Alias.get_by(email=email)
  1428:         or Contact.get_by(reply_email=email)
  1429:         or DeletedAlias.get_by(email=email)
  1430:     ):
  1431:         return False
  1432:     return True
=== EXACT body of ModelMixin.get_by() (the .first() with no ORDER BY) ===
  82:     @classmethod
  83:     def get_by(cls, **kw):
  84:         return Session.query(cls).filter_by(**kw).first()
=== EXACT body of is_reverse_alias() [app/email_utils.py] ===
  1156: def is_reverse_alias(address: str) -> bool:
  1157:     # to take into account the new reverse-alias that doesn't start with "ra+"
  1158:     if Contact.get_by(reply_email=address):
  1159:         return True
  1160:
  1161:     return address.endswith(f"@{config.EMAIL_DOMAIN}") and (
  1162:         address.startswith("reply+") or address.startswith("ra+")
  1163:     )
=== guard bypass: direct Contact.create() ignores available_sl_email() ===
before any contact: available_sl_email('dupe-guard@sl.local') = True
after 1st Contact.create: available_sl_email('dupe-guard@sl.local') = False (guard would now block the GENERATOR from choosing this value)
direct 2nd Contact.create with SAME reply_email SUCCEEDED: c1.id=1 c2.id=2; rows sharing reply_email=2
=> available_sl_email() is a generation-time TOCTOU check only; a direct write bypasses it and no DB UNIQUE constraint prevents the duplicate.
DONE
```

- **Exactly three checks.** `available_sl_email(email)` returns `False` if **any** of `Alias.get_by(email=email)`, `Contact.get_by(reply_email=email)`, or `DeletedAlias.get_by(email=email)` matches, else `True` (`app/models.py:1425-1432`). There is no fourth, elided check — the earlier report's `or ...` paraphrase is corrected here to the verbatim body.
- **`is_reverse_alias()` has two clauses** (`app/email_utils.py:1156-1163`): it returns `True` if `Contact.get_by(reply_email=address)` matches, **otherwise** returns `address.endswith(f"@{config.EMAIL_DOMAIN}") and (address.startswith("reply+") or address.startswith("ra+"))`. Both clauses matter: routing can treat an address as a reverse-alias by suffix/prefix even when no contact row exists.
- **TOCTOU + bypass.** `available_sl_email('dupe-guard@sl.local')` is `True` before any contact, then `False` after the first `Contact.create` — i.e., the guard *would* stop the **generator** from re-choosing that value. But a **direct** second `Contact.create(...)` with the **same** `reply_email` **succeeds** (`c1.id=1 c2.id=2; rows sharing reply_email=2`). The guard is only consulted inside the generation loop and is read-then-act (non-atomic); it is **not** backed by a DB constraint, so any non-generator write path (or a concurrent generator, `[inferred]`) can create the duplicate. This is the proof — via a **duplicate seed**, not a reversed chronology — that direct creation bypasses the guard.

**Call sites of `available_sl_email()` (enumerated, not assumed to be only the generator):**

```
===== available_sl_email call sites (grep) =====
app/email_utils.py:1150:        if available_sl_email(reply_email):
app/models.py:1458:    if available_sl_email(random_email):
app/models.py:1706:            if available_sl_email(email):

===== email_handler.py:1163,1185 (TO/CC replace conditions) =====
            # return 421 so the client can retry later
            return False, status.E402

    Session.commit()

    recipient_name = get_alias_recipient_name(alias)
    if recipient_name.message:
        LOG.d(recipient_name.message)
    LOG.d("From header is %s", recipient_name.name)
    add_or_replace_header(msg, headers.FROM, recipient_name.name)

    try:
        if str(msg[headers.TO]).lower() == "undisclosed-recipients:;":
            # no need to replace TO header
            LOG.d("email is sent in BCC mode")
        else:
            replace_header_when_reply(msg, alias, headers.TO)

        replace_header_when_reply(msg, alias, headers.CC)
    except NonReverseAliasInReplyPhase as e:
        LOG.w("non reverse-alias in reply %s %s %s", e, contact, alias)

        # the email is ignored, delete the email log
```

- `available_sl_email()` is called from **three** places: the reverse-alias generator `generate_reply_email()` (`app/email_utils.py:1150`), random-alias generation (`app/models.py:1458`), and custom-alias generation (`app/models.py:1706`). The claim is therefore scoped precisely: within the **reply-email creation path**, uniqueness is guarded only by the `email_utils.py:1150` call — and only best-effort, as shown above. (The appended `email_handler.py` excerpt shows the `TO`/`CC` replace conditions relevant to §8.3.)

#### 6.2.1 Genuine concurrent reproduction of the TOCTOU race (two processes: real check → real commit)

§6.2 above proves the guard is bypassable by a *single* direct write. This subsection proves the stronger claim behind the wrong-user root cause: **two independent OS processes, each running the real `available_sl_email()` → `Contact.create()` → `Session.commit()` path concurrently, both pass the check and both commit**, leaving two `Contact` rows that share one `reply_email` and belong to **two different users**. No product code is modified — the harness (`observe_race.py`, embedded verbatim in §11.3) only *drives* and *observes* the real functions.

**What is real vs. what is a control (stated honestly).** The following are the **real application code** under test, unmodified: `available_sl_email()` (`app/models.py:1425-1432`), `generate_reply_email()` (`app/email_utils.py:1103`), `Contact.create()` + `Session.commit()` (the ORM write path), and the schema (no `UNIQUE` on `reply_email`). The following are **synchronization controls** the harness adds *around* that real code — they make the check-before-commit interleaving deterministically observable, they do **not** fabricate the defect:

- a **filesystem barrier**: each process writes a `<role>.checked` marker immediately after its `available_sl_email()` call and waits until *both* markers exist before either calls `Session.commit()`. This forces the "both read stale → both write" interleaving that in production occurs only opportunistically when two windows overlap.
- a **leader/follower hand-off** in `gen` mode: because `random_string()` builds each candidate with `secrets.choice()` — a CSPRNG backed by `os.urandom` (`app/utils.py:41-47`) — two *independent* `generate_reply_email()` calls essentially never choose the same candidate, and `random.seed()` cannot force a collision. The harness therefore has **p1 mint one reverse-alias with the real `generate_reply_email()` loop and publish it**, and **p2 consume that generator-produced value** — presenting one *genuinely generator-produced* value to both concurrent writers. The hand-off substitutes for a natural generator collision (which is [inferred] astronomically unlikely — see bounds below); everything downstream of the value (the check, the create, the commit) is real and concurrent.

**Commands** (seed once; then two concurrent `attempt` processes share one barrier dir; then `report`). Each attempt below uses a fresh seed and a fresh barrier dir:

```
# --- fixed-target concurrency (attempt 1: barrier /tmp/racebarA1) ---
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_race.py seed'
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_race.py attempt p1 /tmp/racebarA1 fixed 0' &
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_race.py attempt p2 /tmp/racebarA1 fixed 0' &
wait
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_race.py report /tmp/racebarA1'
# --- attempt 2 repeats the same three steps with a fresh seed + barrier /tmp/racebarA2 ---
# --- generator-path collision (barrier /tmp/racebarG): identical steps but mode 'gen' (p1 mints, p2 consumes) ---
```

**Output — attempt 1 (fixed target, complete and unedited):**

```
===== SEED =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:47:58,882 - SL - DEBUG - 16222 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 02:48:03,474 - SL - INFO - 16222 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 02:48:03,477 - SL - INFO - 16222 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 02:48:03,478 - SL - INFO - 16222 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 02:48:03,479 - SL - DEBUG - 16222 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 02:48:03,762 - SL - INFO - 16222 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:04,024 - SL - INFO - 16222 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:04,039 - SL - DEBUG - 16222 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_test553@sl.local
2026-07-14 02:48:04,046 - SL - INFO - 16222 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:04,056 - SL - DEBUG - 16222 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_list866@sl.local
2026-07-14 02:48:04,064 - SL - INFO - 16222 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== SEED (race) ===
user_a.id=1 first_alias_a.id=1 email='simplelogin-newsletter.test415@sl.local'
user_b.id=2 first_alias_b.id=2 email='simplelogin-newsletter.word912@sl.local'
contacts sharing 'race-shared@sl.local' at seed = 0
SEED DONE
===== p1 (concurrent) =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:05,309 - SL - DEBUG - 16250 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE attempt role=p1 mode=fixed ===
[p1] target='race-shared@sl.local'
[p1] available_sl_email(target) BEFORE barrier = True
[p1] barrier: wrote p1.checked; waiting for peer to also check ...
[p1] barrier: both processes checked = True - proceeding to Contact.create + commit
[p1] Contact.create(reply_email=target, alias_id=1, website='racea@nowhere.net') committed id=2
[p1] post-commit rows sharing target = 2
RACE attempt DONE role=p1
===== p2 (concurrent) =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:05,274 - SL - DEBUG - 16251 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE attempt role=p2 mode=fixed ===
[p2] target='race-shared@sl.local'
[p2] available_sl_email(target) BEFORE barrier = True
[p2] barrier: wrote p2.checked; waiting for peer to also check ...
[p2] barrier: both processes checked = True - proceeding to Contact.create + commit
[p2] Contact.create(reply_email=target, alias_id=2, website='raceb@nowhere.net') committed id=1
[p2] post-commit rows sharing target = 2
RACE attempt DONE role=p2
===== REPORT =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:07,395 - SL - DEBUG - 16283 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE report target='race-shared@sl.local' ===
rows sharing reply_email='race-shared@sl.local': count=2
  ctid=(0,1) id=1 alias_id=2 user_id=2 website='raceb@nowhere.net'
  ctid=(0,2) id=2 alias_id=1 user_id=1 website='racea@nowhere.net'
p1.checked target : race-shared@sl.local
p2.checked target : race-shared@sl.local
p1.committed      : checked=True outcome=OK(id=2) postcount=2
p2.committed      : checked=True outcome=OK(id=1) postcount=2
VERDICT: both_processes_saw_available=True: True; duplicate_reply_email_committed: True (no UNIQUE on reply_email blocked the second insert)
RACE report DONE
```

**Output — attempt 2 (fixed target, fresh seed + barrier, complete and unedited):**

```
===== SEED =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:09,516 - SL - DEBUG - 16313 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 02:48:14,326 - SL - INFO - 16313 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 02:48:14,329 - SL - INFO - 16313 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 02:48:14,331 - SL - INFO - 16313 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 02:48:14,332 - SL - DEBUG - 16313 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 02:48:14,617 - SL - INFO - 16313 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:14,882 - SL - INFO - 16313 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:14,896 - SL - DEBUG - 16313 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_word990@sl.local
2026-07-14 02:48:14,903 - SL - INFO - 16313 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:14,912 - SL - DEBUG - 16313 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_list569@sl.local
2026-07-14 02:48:14,919 - SL - INFO - 16313 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== SEED (race) ===
user_a.id=1 first_alias_a.id=1 email='simplelogin-newsletter.list749@sl.local'
user_b.id=2 first_alias_b.id=2 email='simplelogin-newsletter.word640@sl.local'
contacts sharing 'race-shared@sl.local' at seed = 0
SEED DONE
===== p1 (concurrent) =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:16,096 - SL - DEBUG - 16340 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE attempt role=p1 mode=fixed ===
[p1] target='race-shared@sl.local'
[p1] available_sl_email(target) BEFORE barrier = True
[p1] barrier: wrote p1.checked; waiting for peer to also check ...
[p1] barrier: both processes checked = True - proceeding to Contact.create + commit
[p1] Contact.create(reply_email=target, alias_id=1, website='racea@nowhere.net') committed id=1
[p1] post-commit rows sharing target = 1
RACE attempt DONE role=p1
===== p2 (concurrent) =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:16,068 - SL - DEBUG - 16341 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE attempt role=p2 mode=fixed ===
[p2] target='race-shared@sl.local'
[p2] available_sl_email(target) BEFORE barrier = True
[p2] barrier: wrote p2.checked; waiting for peer to also check ...
[p2] barrier: both processes checked = True - proceeding to Contact.create + commit
[p2] Contact.create(reply_email=target, alias_id=2, website='raceb@nowhere.net') committed id=2
[p2] post-commit rows sharing target = 2
RACE attempt DONE role=p2
===== REPORT =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:18,223 - SL - DEBUG - 16372 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE report target='race-shared@sl.local' ===
rows sharing reply_email='race-shared@sl.local': count=2
  ctid=(0,1) id=1 alias_id=1 user_id=1 website='racea@nowhere.net'
  ctid=(0,2) id=2 alias_id=2 user_id=2 website='raceb@nowhere.net'
p1.checked target : race-shared@sl.local
p2.checked target : race-shared@sl.local
p1.committed      : checked=True outcome=OK(id=1) postcount=1
p2.committed      : checked=True outcome=OK(id=2) postcount=2
VERDICT: both_processes_saw_available=True: True; duplicate_reply_email_committed: True (no UNIQUE on reply_email blocked the second insert)
RACE report DONE
```

**Output — generator-path attempt (mode `gen`, leader/follower, complete and unedited):**

```
===== SEED =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:20,204 - SL - DEBUG - 16402 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 02:48:25,445 - SL - INFO - 16402 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 02:48:25,448 - SL - INFO - 16402 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 02:48:25,449 - SL - INFO - 16402 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 02:48:25,451 - SL - DEBUG - 16402 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 02:48:25,735 - SL - INFO - 16402 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:25,999 - SL - INFO - 16402 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:26,013 - SL - DEBUG - 16402 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_list708@sl.local
2026-07-14 02:48:26,022 - SL - INFO - 16402 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 02:48:26,031 - SL - DEBUG - 16402 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_test843@sl.local
2026-07-14 02:48:26,039 - SL - INFO - 16402 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== SEED (race) ===
user_a.id=1 first_alias_a.id=1 email='simplelogin-newsletter.test506@sl.local'
user_b.id=2 first_alias_b.id=2 email='simplelogin-newsletter.word703@sl.local'
contacts sharing 'race-shared@sl.local' at seed = 0
SEED DONE
===== p1 (concurrent) =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:27,219 - SL - DEBUG - 16431 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE attempt role=p1 mode=gen ===
[p1] generate_reply_email(...) -> 'race-contact_at_nowhere_net_tghnhlpphn@sl.local' (real generator; published to peer)
[p1] target='race-contact_at_nowhere_net_tghnhlpphn@sl.local'
[p1] available_sl_email(target) BEFORE barrier = True
[p1] barrier: wrote p1.checked; waiting for peer to also check ...
[p1] barrier: both processes checked = True - proceeding to Contact.create + commit
[p1] Contact.create(reply_email=target, alias_id=1, website='racea@nowhere.net') committed id=2
[p1] post-commit rows sharing target = 2
RACE attempt DONE role=p1
===== p2 (concurrent) =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:27,216 - SL - DEBUG - 16432 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE attempt role=p2 mode=gen ===
[p2] consumed generator-produced target -> 'race-contact_at_nowhere_net_tghnhlpphn@sl.local'
[p2] target='race-contact_at_nowhere_net_tghnhlpphn@sl.local'
[p2] available_sl_email(target) BEFORE barrier = True
[p2] barrier: wrote p2.checked; waiting for peer to also check ...
[p2] barrier: both processes checked = True - proceeding to Contact.create + commit
[p2] Contact.create(reply_email=target, alias_id=2, website='raceb@nowhere.net') committed id=1
[p2] post-commit rows sharing target = 2
RACE attempt DONE role=p2
===== REPORT =====
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 02:48:29,325 - SL - DEBUG - 16463 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
=== RACE report target='race-contact_at_nowhere_net_tghnhlpphn@sl.local' ===
rows sharing reply_email='race-contact_at_nowhere_net_tghnhlpphn@sl.local': count=2
  ctid=(0,1) id=1 alias_id=2 user_id=2 website='raceb@nowhere.net'
  ctid=(0,2) id=2 alias_id=1 user_id=1 website='racea@nowhere.net'
p1.checked target : race-contact_at_nowhere_net_tghnhlpphn@sl.local
p2.checked target : race-contact_at_nowhere_net_tghnhlpphn@sl.local
p1.committed      : checked=True outcome=OK(id=2) postcount=2
p2.committed      : checked=True outcome=OK(id=1) postcount=2
VERDICT: both_processes_saw_available=True: True; duplicate_reply_email_committed: True (no UNIQUE on reply_email blocked the second insert)
RACE report DONE
```

**Analysis of the three attempts:**

- **All three attempts, identical verdict.** In each attempt both processes' `available_sl_email(target)` returned `True` *before* either committed — the check acquires no row lock (`app/models.py:1425-1432`) — and both `Contact.create`+`Session.commit()` **succeeded**, leaving `count=2` rows sharing the one `reply_email`. Every attempt prints `VERDICT: both_processes_saw_available=True: True; duplicate_reply_email_committed: True`.
- **The two duplicate rows belong to different users.** Every `report` shows one row with `user_id=1` (alias_a, `racea@nowhere.net`) and one with `user_id=2` (alias_b, `raceb@nowhere.net`). This is exactly the two-different-users precondition that §7 shows drives the wrong-user forward: once both rows exist, `Contact.get_by(reply_email=...)` → `.first()` (§6.3) picks one arbitrarily and the reply follows *that* row's `alias.user` (§6.4).
- **Non-deterministic id/interleave, honestly shown.** The committed `id`s flip between runs (attempt 1: p1→`id=2`, p2→`id=1`; attempt 2: p1→`id=1`, p2→`id=2`) because `RESTART IDENTITY` resets the sequence each seed and whichever `commit()` lands first takes the lower id; and a process can observe `postcount=1` transiently if it commits just before its peer (attempt 2, p1 `postcount=1`), while the *final* `report` count is `2` in every attempt. These variations are the expected signature of a genuine concurrent interleaving, not a scripted sequence.
- **Generator path proven, not simulated.** In `gen` mode the shared value `race-contact_at_nowhere_net_tghnhlpphn@sl.local` was **produced by the real `generate_reply_email()` loop** (p1: `generate_reply_email(...) -> '…tghnhlpphn@sl.local' (real generator; published to peer)`; p2: `consumed generator-produced target -> '…tghnhlpphn@sl.local'`). The value differs run-to-run (an earlier gen attempt produced `…vjlsjvlbts@sl.local`) because `secrets.choice` is non-deterministic; the invariant — both writers commit the same generator-produced value — holds regardless.

**Observed vs. inferred bounds (honesty pass for the race claim):**

- **Observed (directly reproduced, 3/3 attempts):** with the check-before-commit interleaving realized, the real code path commits two rows sharing one `reply_email` for two different users. The enabling conditions are real and independently verified elsewhere in this document: the check takes no lock (`app/models.py:1425-1432`) and no `UNIQUE`/exclusion constraint exists on `reply_email` (§6.1; migration `2021_071310_78403c7b8089`:22 creates `ix_contact_reply_email` with `unique=False`).
- **Inferred (labeled [inferred]):** in production the interleaving is *not* externally synchronized, so the hazard's natural frequency is bounded by (a) the width of each writer's read→commit window and (b), for the **generator** path specifically, the probability that two independent `generate_reply_email()` calls pick the same candidate — which, given `secrets.choice` over the candidate space (`app/utils.py:41-47`), is astronomically small. A *natural* generator collision could not be forced without the leader/follower hand-off, so that specific frequency claim is [inferred] and bounded by the CSPRNG candidate space. The concurrent hazard is, by contrast, **directly reproducible for any non-generator or repeated-value write path** (a re-used value, a bulk import, an admin/script create), which the fixed-target attempts model.
- **Why the controls do not overstate the finding.** The barrier and the hand-off only *schedule* the interleaving and *supply* one value to two writers; they grant no privilege and skip no validation. Remove them and the same two `commit()`s still succeed whenever the two windows happen to overlap on the same value — because the only thing that could reject the second insert (a DB `UNIQUE` constraint) does not exist. The controls make an opportunistic race deterministically observable; they do not create it.

### 6.3 The lookup returns a row with no application-specified order

`Contact.get_by(reply_email=...)` → `Session.query(Contact).filter_by(reply_email=...).first()` (`app/models.py:82-84`) emits `LIMIT 1` with **no `ORDER BY`**. Per the official semantics (§4.1, §9): with more than one matching row and no `ORDER BY`, SQLAlchemy returns whatever the database yields first, and PostgreSQL returns rows in an **unspecified order** that "*depend[s] on the scan and join plan types and the order on disk*" and "*must not be relied on*." The application thus imposes **no** determinism on which of several same-`reply_email` contacts is chosen. The concrete PostgreSQL determinant for the seeded two-row case (plan + physical `ctid`) is analyzed with `EXPLAIN`/`ctid` evidence in §7.4; the honest bound is stated there and in the coverage pass (§12).

### 6.4 A wrong contact yields a wrong alias, user, mailbox, and persisted log

Because `alias = contact.alias` (`:994`), `user = alias.user` (`:1004`), and `EmailLog.create(..., user_id=contact.user_id, ...)` (`:1042-1050`) all derive from the resolved contact, resolving the *other* same-`reply_email` contact propagates end-to-end: a different alias, a different owning user, a different candidate mailbox, and — on the authorized/fallback path — a log row attributing the reply to the wrong user (`EmailLog.user_id`). §7 shows this propagation directly: flipping only the contacts' insertion order flips the resolved `user_id` from `1` to `2` under the identical input.

### 6.5 The same non-unique lookup gates multiple routing decisions

`Contact.get_by(reply_email=...)` (and its `is_reverse_alias()` wrapper) is reused beyond the primary resolution: the second header-rewrite lookup in `replace_header_when_reply()` (`email_handler.py:364`, exercised in §8.3), and the routing/bounce predicates in `handle()` (`is_reverse_alias()` is called at `email_handler.py:2166` and `:2195`). Every one of these decisions inherits the same non-unique, unordered `.first()` semantics. Direct `Contact.get_by`/`is_reverse_alias` calls used purely to corroborate are labeled `[non-canonical]`; the routing itself is exercised canonically through `handle()` in §4.3.

### 6.6 Normalization widens the *inbound* match surface (but does not, by itself, create multi-row matches)

`normalize_reply_email()` is applied to the **inbound** `rcpt_to` only; the stored `Contact.reply_email` values are compared by **exact equality** in the query. So multiple inbound *spellings* collapse onto **one** stored key (many-to-one on input), but a **multi-row** match still requires **more than one stored row** holding that same exact normalized key. This corrects the earlier conflation; the runtime exact-equality proof is in §8.2.

### 6.7 Forward-phase minting (where a contact's `reply_email` originates)

In the forward phase, a contact's `reply_email` is minted by `generate_reply_email(...)` (`app/email_utils.py:1103`), which loops choosing a candidate and calling `available_sl_email(reply_email)` (`app/email_utils.py:1150`) until it passes — the best-effort guard analyzed in §6.2. The official reverse-alias contract intends this value to be unique per `(alias, contact)` pair (§9); the schema does not enforce that intent, which is the root permissiveness this investigation documents.

---
## 7. Cross-event behavior (same unchanged input, every run disclosed)

### 7.0 Methodology (seed once, drive the identical input many times; N=100 x two runs per condition)

The question "does the same reply resolve differently over time?" requires driving the **same unchanged input against the same persisted rows** - not regenerating users/aliases/contacts/IDs per run. `observe_dist.py` therefore separates **seed** from **drive**:

- **`seed <ab|ba> <on|off>`** truncates to a clean slate, seeds a **fixed** dataset (user_A `usera@mailbox.test`, user_B `userb@mailbox.test`, one alias each, and **two** `Contact` rows sharing `reply_email='dist-shared@sl.local'` with fixed `website_email`s `a-dist@nowhere.net` / `b-dist@nowhere.net`), sets both aliases' `disable_email_spoofing_check` per `on|off`, prints the physical `ctid` order and `EXPLAIN` of the exact lookup, then **stops**. `ab` vs `ba` controls only the contacts' **insertion order**.
- **`drive <N> <mid_prefix>`** does **not** reseed. It asserts exactly two contacts share the `reply_email` (else exits `3`), then drives the **canonical** `email_handler.handle_reply()` **`N=100`** times with an **identical** message. Every field is a **fixed constant** - `mail_from='usera@mailbox.test'` (user_A's mailbox), `rcpt_to`/`To='dist-shared@sl.local'`, `From='someone@nowhere.net'`, `Subject='dist reply'`, and a fixed `Date`/body - so that **only the Message-ID varies** (`<mid_prefix>-i@sl.local`), which is required only to give each event its own distinct `EmailLog` row for correlation. (This corrects an earlier harness in which the `Subject` also varied per event; here `Subject` is the fixed constant `'dist reply'`.)
- **Per-event correlation and five distributions.** Each event is correlated to its own `EmailLog` by **exact Message-ID** (`EmailLog.get_by(message_id=mid)`), capturing `contact_id`/`alias_id`/`user_id`/`mailbox_id`/`is_reply` and the return `code`. After the loop the harness prints, as one contiguous block, five aggregates over all `N=100` events: (1) **status** distribution by return code; (2) the **full keyed** distribution over the resolved `EmailLog` tuple (with an explicit `sum == N` check); (3) the **OUTBOUND** distribution of stored `SendRequest`s grouped by `envelope_to` (captured via `mail_sender.store_emails_instead_of_sending(True)`, so no external SMTP is used); (4) the **NOTIFICATION** distribution of new `SentAlert` rows created during the drive; and (5) per-event `handle_reply()` **durations** (monotonic clock: total/mean/min/max). For readability the harness also prints the first three events and the last event as a **sample**; the aggregate counters are the authoritative full distribution.
- **Contiguous, framework-noise-free report.** The application emits its own framework logging (timestamps, PIDs, random alias names, DMARC/rate-limit lines) to stdout *during* the drive loop. To make the observational output verifiable as a single unedited unit, `do_drive()` **buffers** its entire structured report into a list and flushes it with one `print()` **after** the loop, so the block from `=== DRIVE mode` to `DRIVE DONE` is guaranteed contiguous and free of interleaved framework lines. Each block below was extracted as exactly that contiguous span; the preceding framework logging (non-deterministic timestamps/PIDs/random names) is elided as noise.
- **Machine-verified reproducibility across the two runs.** For each condition the two independent drive blocks are compared after masking only the three legitimately-varying tokens - the `mid_prefix`, the per-event `Message-ID`, and the wall-clock **duration** line. The masked block is split into two dimensions and SHA256-hashed: **RESOLUTION** (header + sample events + status + full keyed `EmailLog`) and **NOTIF/OUTBOUND** (the outbound + `SentAlert` aggregates). This lets the resolution outcome (the crux of the wrong-user question) be certified byte-identical across runs independently of the notification side effects, which - as section 7.2 shows - are legitimately rate-limited.
- Argument bounds are validated (`order in {ab,ba}`, `spoof in {on,off}`, `N in 1..1000`, `mid_prefix` matches `[A-Za-z0-9_.-]+`), with non-zero exits on invalid input, and the harness refuses to run unless `DB_URI` is the canonical test DSN and the connected database name is `test`.

**Every run performed against the clean-slate database is disclosed below** - four seeds and eight drive runs (each of the four conditions driven twice, each drive a fresh process at `N=100`) - none omitted. The four conditions are the cross product of insertion order (`ab`/`ba`) and spoof control (`off` = **default**; `on` = **`[non-canonical]` fallback**).

The canonical runner used for every invocation:

```
# Canonical runner (host shell). ARGS is one of the argument strings listed per
# condition below, e.g. "seed ab off" or "drive 100 aboffr1". Each drive run is a
# FRESH process (fresh Python interpreter + DB session) against the SAME seeded rows.
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_dist.py '"$ARGS"' 2>&1'
```

### 7.1 Condition `ab` / `off` - default control, sender owns the first-inserted contact (correct user)

Argument strings (three fresh processes): `"seed ab off"`, then `"drive 100 aboffr1"`, then `"drive 100 aboffr2"`.

**Seed** - canonical block, complete and unedited (`ARGS="seed ab off"`):

```
=== SEED mode order=ab spoof=off (disable_email_spoofing_check=False) ===
shared reply_email='dist-shared@sl.local'  FIXED_MAIL_FROM='usera@mailbox.test'
user_a.id=1 alias_a.id=3  user_b.id=2 alias_b.id=4
insertion #1 -> Contact id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
insertion #2 -> Contact id=2 alias_id=4 user_id=2 website='b-dist@nowhere.net'
--- physical order (ctid) of rows sharing reply_email ---
  ctid=(0,1) id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
  ctid=(0,2) id=2 alias_id=4 user_id=2 website='b-dist@nowhere.net'
--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---
  Limit  (cost=0.14..4.16 rows=1 width=4)
    ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..4.16 rows=1 width=4)
          Index Cond: ((reply_email)::text = 'dist-shared@sl.local'::text)
--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---
  Limit  (cost=0.00..11.00 rows=1 width=4)
    ->  Seq Scan on contact  (cost=0.00..11.00 rows=1 width=4)
          Filter: ((reply_email)::text = 'dist-shared@sl.local'::text)
SEED DONE (persisted; run 'drive' next)
```

First-inserted `Contact id=1` (`user_A`) is at `ctid=(0,1)`; second at `ctid=(0,2)`. **Drive run 1** (`ARGS="drive 100 aboffr1"`), complete and unedited buffered report:

```
=== DRIVE mode N=100 mid_prefix='aboffr1' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
driving handle_reply() 100x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local', From='someone@nowhere.net', Subject='dist reply', Date/body FIXED; ONLY Message-ID varies
  event mid=<aboffr1-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<aboffr1-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<aboffr1-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<aboffr1-99@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
--- distribution by return code (status) ---
  code='250 Message accepted for delivery': 100
--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---
  code='250 Message accepted for delivery'|EmailLog:contact_id=1,alias_id=3,user_id=1,mailbox_id=1,is_reply=True: 100
  (sum of keyed distribution = 100; expected N = 100; match = True)
--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---
  total stored SendRequest = 100
  envelope_to='a-dist@nowhere.net': 100
--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---
  total new SentAlert = 0
  (none)
--- per-event handle_reply() durations (monotonic) ---
  events=100 total=2424.6 ms mean=24.25 ms min=21.76 ms max=98.37 ms
DRIVE DONE
```

**Drive run 2** (`ARGS="drive 100 aboffr2"`, same persisted rows, fresh process), complete and unedited:

```
=== DRIVE mode N=100 mid_prefix='aboffr2' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
driving handle_reply() 100x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local', From='someone@nowhere.net', Subject='dist reply', Date/body FIXED; ONLY Message-ID varies
  event mid=<aboffr2-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<aboffr2-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<aboffr2-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<aboffr2-99@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
--- distribution by return code (status) ---
  code='250 Message accepted for delivery': 100
--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---
  code='250 Message accepted for delivery'|EmailLog:contact_id=1,alias_id=3,user_id=1,mailbox_id=1,is_reply=True: 100
  (sum of keyed distribution = 100; expected N = 100; match = True)
--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---
  total stored SendRequest = 100
  envelope_to='a-dist@nowhere.net': 100
--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---
  total new SentAlert = 0
  (none)
--- per-event handle_reply() durations (monotonic) ---
  events=100 total=2410.2 ms mean=24.10 ms min=21.51 ms max=90.59 ms
DRIVE DONE
```

Machine-verification of run 1 vs run 2 (masking only `mid_prefix`, per-event Message-ID, and the duration line):

```
RESOLUTION (header+samples+status+keyed EmailLog)  run1 sha256 = 7a894de53f06319bb4eea6391a526e68dc136f997708946bc8adc67c90eebd68
                                                    run2 sha256 = 7a894de53f06319bb4eea6391a526e68dc136f997708946bc8adc67c90eebd68   -> IDENTICAL
NOTIF/OUTBOUND (outbound + SentAlert)               run1 sha256 = 79d864ef48a494cda37c686eff3460b5d3a144b3fc32d5ba0f81046331187fb0
                                                    run2 sha256 = 79d864ef48a494cda37c686eff3460b5d3a144b3fc32d5ba0f81046331187fb0   -> IDENTICAL
```

**Result:** both runs **100/100** `code='250 Message accepted for delivery'`, all resolving `contact_id=1,alias_id=3,user_id=1,mailbox_id=1` with **100** outbound `SendRequest`s to `a-dist@nowhere.net` and **0** `SentAlert`s - the correct owner (user_A). The RESOLUTION and NOTIF/OUTBOUND dimensions are byte-identical across the two runs.

### 7.2 Condition `ba` / `off` - default control, `.first()` resolves user_B -> **E214 access-control rejection (no forward)**

Argument strings: `"seed ba off"`, then `"drive 100 baoffr1"`, then `"drive 100 baoffr2"`. Insertion order is reversed, so `Contact id=1` is now `user_B`'s (at `ctid=(0,1)`).

**Seed** - complete and unedited (`ARGS="seed ba off"`):

```
=== SEED mode order=ba spoof=off (disable_email_spoofing_check=False) ===
shared reply_email='dist-shared@sl.local'  FIXED_MAIL_FROM='usera@mailbox.test'
user_a.id=1 alias_a.id=3  user_b.id=2 alias_b.id=4
insertion #1 -> Contact id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
insertion #2 -> Contact id=2 alias_id=3 user_id=1 website='a-dist@nowhere.net'
--- physical order (ctid) of rows sharing reply_email ---
  ctid=(0,1) id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
  ctid=(0,2) id=2 alias_id=3 user_id=1 website='a-dist@nowhere.net'
--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---
  Limit  (cost=0.14..4.16 rows=1 width=4)
    ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..4.16 rows=1 width=4)
          Index Cond: ((reply_email)::text = 'dist-shared@sl.local'::text)
--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---
  Limit  (cost=0.00..11.00 rows=1 width=4)
    ->  Seq Scan on contact  (cost=0.00..11.00 rows=1 width=4)
          Filter: ((reply_email)::text = 'dist-shared@sl.local'::text)
SEED DONE (persisted; run 'drive' next)
```

**Drive run 1** (`ARGS="drive 100 baoffr1"`), identical `mail_from='usera@mailbox.test'`, complete and unedited:

```
=== DRIVE mode N=100 mid_prefix='baoffr1' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
driving handle_reply() 100x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local', From='someone@nowhere.net', Subject='dist reply', Date/body FIXED; ONLY Message-ID varies
  event mid=<baoffr1-0@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (no EmailLog persisted on this path)
  event mid=<baoffr1-1@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (no EmailLog persisted on this path)
  event mid=<baoffr1-2@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (no EmailLog persisted on this path)
  event mid=<baoffr1-99@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (no EmailLog persisted on this path)
--- distribution by return code (status) ---
  code='250 SL E214 Unauthorized for using reverse alias': 100
--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---
  code='250 SL E214 Unauthorized for using reverse alias'|EmailLog=None: 100
  (sum of keyed distribution = 100; expected N = 100; match = True)
--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---
  total stored SendRequest = 4
  envelope_to='userb@mailbox.test': 4
--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---
  total new SentAlert = 4
  alert_type='reverse_alias_unknown_mailbox',to_email='userb@mailbox.test',user_id=2: 4
--- per-event handle_reply() durations (monotonic) ---
  events=100 total=1924.5 ms mean=19.24 ms min=17.41 ms max=46.60 ms
DRIVE DONE
```

**Drive run 2** (`ARGS="drive 100 baoffr2"`, same persisted rows, fresh process), complete and unedited:

```
=== DRIVE mode N=100 mid_prefix='baoffr2' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
driving handle_reply() 100x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local', From='someone@nowhere.net', Subject='dist reply', Date/body FIXED; ONLY Message-ID varies
  event mid=<baoffr2-0@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (no EmailLog persisted on this path)
  event mid=<baoffr2-1@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (no EmailLog persisted on this path)
  event mid=<baoffr2-2@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (no EmailLog persisted on this path)
  event mid=<baoffr2-99@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (no EmailLog persisted on this path)
--- distribution by return code (status) ---
  code='250 SL E214 Unauthorized for using reverse alias': 100
--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---
  code='250 SL E214 Unauthorized for using reverse alias'|EmailLog=None: 100
  (sum of keyed distribution = 100; expected N = 100; match = True)
--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---
  total stored SendRequest = 0
  (none)
--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---
  total new SentAlert = 0
  (none)
--- per-event handle_reply() durations (monotonic) ---
  events=100 total=1924.6 ms mean=19.25 ms min=17.59 ms max=33.18 ms
DRIVE DONE
```

Machine-verification of run 1 vs run 2:

```
RESOLUTION (header+samples+status+keyed EmailLog)  run1 sha256 = d4b7631337f8e015578369789033484d8b7450f9d612d902e2fd488fee3b37f4
                                                    run2 sha256 = d4b7631337f8e015578369789033484d8b7450f9d612d902e2fd488fee3b37f4   -> IDENTICAL
NOTIF/OUTBOUND (outbound + SentAlert)               run1 sha256 = b6064b3b8bd298237fb7f1c20f014019e827b2d3c82d0b9d78677b7e9eab57bf
                                                    run2 sha256 = 98805abcd2ab127510adf93bbc3bd89de00749c68fa55a15f4ca5db0f29f6776   -> DIFFERS
```

**Result:** both runs **100/100** `code='250 SL E214 Unauthorized for using reverse alias'` with `EmailLog=None` - the **RESOLUTION dimension is byte-identical** across runs. The identical input resolves `user_B`'s contact (via `.first()` at `ctid=(0,1)`); user_A's sending mailbox is **not** authorized for user_B's alias, so under the **default** spoof control the reply is **rejected with E214 before any `EmailLog`/forward** (`handle_unknown_mailbox()` path). Under default settings, wrong-user *resolution* therefore manifests as an **access-control boundary**, not a wrong-user delivery.

**The NOTIF/OUTBOUND dimension legitimately DIFFERS between the two runs, and this difference is itself a security-relevant finding** (see section 5/section 8 and the security analysis): run 1 emitted **4** `reverse_alias_unknown_mailbox` alert emails to `userb@mailbox.test` (user_B, `user_id=2`) and created **4** `SentAlert` rows; run 2, executed against the **same persisted dataset without reseeding**, emitted **0** because SimpleLogin's `send_email_with_rate_control()` caps this alert at **4 per recipient per `alert_type` per 24h** and the `SentAlert` rows from run 1 persist and suppress the repeat. The exact difference, as computed by `diff` over the masked NOTIF/OUTBOUND blocks:

```
2,3c2,3
<   total stored SendRequest = 4
<   envelope_to='userb@mailbox.test': 4
---
>   total stored SendRequest = 0
>   (none)
5,6c5,6
<   total new SentAlert = 4
<   alert_type='reverse_alias_unknown_mailbox',to_email='userb@mailbox.test',user_id=2: 4
---
>   total new SentAlert = 0
>   (none)
```

Two points follow, reported honestly rather than smoothed over: (a) the **resolution/status** outcome - the answer to the wrong-user question - is fully stable and byte-identical across runs (100/100 E214, `EmailLog=None`); (b) the **notification side effect is rate-limited and stateful**, so it is *not* reproducible run-to-run on the same dataset, and the alert is delivered to **user_B** (the wrongly-resolved contact's owner), **not** to user_A who actually sent the reply - a cross-user information-disclosure consequence of the same ordering-dependent `.first()` resolution.

### 7.3 Conditions `ab`/`on` and `ba`/`on` - `[non-canonical]` fallback (spoof check disabled) -> the actual wrong-user forward

With `disable_email_spoofing_check = True` on both aliases (a **non-default** per-alias flag; labeled `[non-canonical]` fallback because it bypasses the default access-control gate), the unauthorized-mailbox branch is skipped and the reply proceeds under the resolved contact's user.

**`ab`/`on`** - argument strings `"seed ab on"`, `"drive 100 abonr1"`, `"drive 100 abonr2"`.

**Seed** - complete and unedited:

```
=== SEED mode order=ab spoof=on (disable_email_spoofing_check=True) ===
shared reply_email='dist-shared@sl.local'  FIXED_MAIL_FROM='usera@mailbox.test'
user_a.id=1 alias_a.id=3  user_b.id=2 alias_b.id=4
insertion #1 -> Contact id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
insertion #2 -> Contact id=2 alias_id=4 user_id=2 website='b-dist@nowhere.net'
--- physical order (ctid) of rows sharing reply_email ---
  ctid=(0,1) id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
  ctid=(0,2) id=2 alias_id=4 user_id=2 website='b-dist@nowhere.net'
--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---
  Limit  (cost=0.14..4.16 rows=1 width=4)
    ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..4.16 rows=1 width=4)
          Index Cond: ((reply_email)::text = 'dist-shared@sl.local'::text)
--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---
  Limit  (cost=0.00..11.00 rows=1 width=4)
    ->  Seq Scan on contact  (cost=0.00..11.00 rows=1 width=4)
          Filter: ((reply_email)::text = 'dist-shared@sl.local'::text)
SEED DONE (persisted; run 'drive' next)
```

**Drive run 1** (`ARGS="drive 100 abonr1"`), complete and unedited:

```
=== DRIVE mode N=100 mid_prefix='abonr1' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
driving handle_reply() 100x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local', From='someone@nowhere.net', Subject='dist reply', Date/body FIXED; ONLY Message-ID varies
  event mid=<abonr1-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonr1-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonr1-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonr1-99@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
--- distribution by return code (status) ---
  code='250 Message accepted for delivery': 100
--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---
  code='250 Message accepted for delivery'|EmailLog:contact_id=1,alias_id=3,user_id=1,mailbox_id=1,is_reply=True: 100
  (sum of keyed distribution = 100; expected N = 100; match = True)
--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---
  total stored SendRequest = 100
  envelope_to='a-dist@nowhere.net': 100
--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---
  total new SentAlert = 0
  (none)
--- per-event handle_reply() durations (monotonic) ---
  events=100 total=2570.0 ms mean=25.70 ms min=22.09 ms max=114.39 ms
DRIVE DONE
```

**Drive run 2** (`ARGS="drive 100 abonr2"`), complete and unedited:

```
=== DRIVE mode N=100 mid_prefix='abonr2' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
driving handle_reply() 100x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local', From='someone@nowhere.net', Subject='dist reply', Date/body FIXED; ONLY Message-ID varies
  event mid=<abonr2-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonr2-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonr2-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonr2-99@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
--- distribution by return code (status) ---
  code='250 Message accepted for delivery': 100
--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---
  code='250 Message accepted for delivery'|EmailLog:contact_id=1,alias_id=3,user_id=1,mailbox_id=1,is_reply=True: 100
  (sum of keyed distribution = 100; expected N = 100; match = True)
--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---
  total stored SendRequest = 100
  envelope_to='a-dist@nowhere.net': 100
--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---
  total new SentAlert = 0
  (none)
--- per-event handle_reply() durations (monotonic) ---
  events=100 total=2558.3 ms mean=25.58 ms min=21.64 ms max=124.48 ms
DRIVE DONE
```

Machine-verification of run 1 vs run 2:

```
RESOLUTION (header+samples+status+keyed EmailLog)  run1 sha256 = 7a894de53f06319bb4eea6391a526e68dc136f997708946bc8adc67c90eebd68
                                                    run2 sha256 = 7a894de53f06319bb4eea6391a526e68dc136f997708946bc8adc67c90eebd68   -> IDENTICAL
NOTIF/OUTBOUND (outbound + SentAlert)               run1 sha256 = 79d864ef48a494cda37c686eff3460b5d3a144b3fc32d5ba0f81046331187fb0
                                                    run2 sha256 = 79d864ef48a494cda37c686eff3460b5d3a144b3fc32d5ba0f81046331187fb0   -> IDENTICAL
```

**`ba`/`on`** - the wrong-user forward. Argument strings `"seed ba on"`, `"drive 100 baonr1"`, `"drive 100 baonr2"`.

**Seed** - complete and unedited:

```
=== SEED mode order=ba spoof=on (disable_email_spoofing_check=True) ===
shared reply_email='dist-shared@sl.local'  FIXED_MAIL_FROM='usera@mailbox.test'
user_a.id=1 alias_a.id=3  user_b.id=2 alias_b.id=4
insertion #1 -> Contact id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
insertion #2 -> Contact id=2 alias_id=3 user_id=1 website='a-dist@nowhere.net'
--- physical order (ctid) of rows sharing reply_email ---
  ctid=(0,1) id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
  ctid=(0,2) id=2 alias_id=3 user_id=1 website='a-dist@nowhere.net'
--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---
  Limit  (cost=0.14..4.16 rows=1 width=4)
    ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..4.16 rows=1 width=4)
          Index Cond: ((reply_email)::text = 'dist-shared@sl.local'::text)
--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---
  Limit  (cost=0.00..11.00 rows=1 width=4)
    ->  Seq Scan on contact  (cost=0.00..11.00 rows=1 width=4)
          Filter: ((reply_email)::text = 'dist-shared@sl.local'::text)
SEED DONE (persisted; run 'drive' next)
```

**Drive run 1** (`ARGS="drive 100 baonr1"`), complete and unedited:

```
=== DRIVE mode N=100 mid_prefix='baonr1' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
driving handle_reply() 100x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local', From='someone@nowhere.net', Subject='dist reply', Date/body FIXED; ONLY Message-ID varies
  event mid=<baonr1-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonr1-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonr1-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonr1-99@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
--- distribution by return code (status) ---
  code='250 Message accepted for delivery': 100
--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---
  code='250 Message accepted for delivery'|EmailLog:contact_id=1,alias_id=4,user_id=2,mailbox_id=2,is_reply=True: 100
  (sum of keyed distribution = 100; expected N = 100; match = True)
--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---
  total stored SendRequest = 100
  envelope_to='b-dist@nowhere.net': 100
--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---
  total new SentAlert = 0
  (none)
--- per-event handle_reply() durations (monotonic) ---
  events=100 total=2664.6 ms mean=26.65 ms min=22.60 ms max=122.92 ms
DRIVE DONE
```

**Drive run 2** (`ARGS="drive 100 baonr2"`), complete and unedited:

```
=== DRIVE mode N=100 mid_prefix='baonr2' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
driving handle_reply() 100x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local', From='someone@nowhere.net', Subject='dist reply', Date/body FIXED; ONLY Message-ID varies
  event mid=<baonr2-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonr2-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonr2-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonr2-99@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
--- distribution by return code (status) ---
  code='250 Message accepted for delivery': 100
--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---
  code='250 Message accepted for delivery'|EmailLog:contact_id=1,alias_id=4,user_id=2,mailbox_id=2,is_reply=True: 100
  (sum of keyed distribution = 100; expected N = 100; match = True)
--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---
  total stored SendRequest = 100
  envelope_to='b-dist@nowhere.net': 100
--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---
  total new SentAlert = 0
  (none)
--- per-event handle_reply() durations (monotonic) ---
  events=100 total=2680.7 ms mean=26.81 ms min=22.64 ms max=128.75 ms
DRIVE DONE
```

Machine-verification of run 1 vs run 2:

```
RESOLUTION (header+samples+status+keyed EmailLog)  run1 sha256 = 83684ca2ce844b85e6b3274b0706e16259d126fa0e887b628bf6393abda9fb4d
                                                    run2 sha256 = 83684ca2ce844b85e6b3274b0706e16259d126fa0e887b628bf6393abda9fb4d   -> IDENTICAL
NOTIF/OUTBOUND (outbound + SentAlert)               run1 sha256 = ef3b13d0935bb513936c4b424c422cbb4b550e4f6b5899aea6e1047243cf031c
                                                    run2 sha256 = ef3b13d0935bb513936c4b424c422cbb4b550e4f6b5899aea6e1047243cf031c   -> IDENTICAL
```

**Result:** `ab`/`on` -> both runs **100/100** E200 resolving `contact_id=1,alias_id=3,user_id=1,mailbox_id=1` with **100** outbound to `a-dist@nowhere.net` (user_A, correct). `ba`/`on` -> both runs **100/100** E200 resolving `contact_id=1,alias_id=4,user_id=2,mailbox_id=2` - **user_B, the wrong user** (not the owner of the sending mailbox) - with **100** outbound `SendRequest`s to `b-dist@nowhere.net` and `EmailLog.user_id=2` recorded for all 100 events per run. Both dimensions are byte-identical across the two runs. This is the condition under which a reply is actually *selected, logged, and enqueued for forwarding under the wrong user*. (`store_emails_instead_of_sending(True)` means no external SMTP hand-off occurs - application acceptance and a stored `SendRequest` only.)

### 7.4 PostgreSQL determinant: physical `ctid` and plan-dependence (bounded, honest interpretation)

Each seed printed the physical `ctid` order and the `EXPLAIN` of the exact lookup `SELECT contact.id FROM contact WHERE contact.reply_email='dist-shared@sl.local' LIMIT 1`:

- **Physical layout.** After a clean-slate truncate + fresh inserts, the two rows occupy the same page in insertion order: the first-inserted row is at `ctid=(0,1)`, the second at `ctid=(0,2)`. In condition `ab` the `(0,1)` row is user_A's; in `ba` it is user_B's. This was byte-for-byte stable across all four seeds above.
- **Plans observed.** The default plan is `Index Scan using ix_contact_reply_email ... Limit rows=1`; with `enable_indexscan/bitmapscan/indexonlyscan` disabled, the same query plans a `Seq Scan on contact ... Limit rows=1`. All four seeds in this run printed the default `Index Scan` at `cost=0.14..4.16` and the forced `Seq Scan` at `cost=0.00..11.00`. The absolute `cost=` figures are PostgreSQL planner **estimates** that depend on table statistics / `ANALYZE` state and can vary run-to-run and across environments - e.g., on a separate earlier re-run the default `Index Scan` upper cost was observed as `8.16` rather than `4.16` - whereas the plan **structure** (`Index Scan` vs forced `Seq Scan`, both `Limit rows=1`) and the physical `ctid` ordering were stable. The interpretation below rests on the plan **structure** and physical order, **not** on the absolute cost estimate.
- **What this does and does not prove.** The application specifies **no `ORDER BY`** (`ModelMixin.get_by()` -> `.filter_by(**kw).first()`, [app/models.py:L82-L84]); PostgreSQL is therefore free to return the matching rows in an unspecified order that, per its documentation, must not be relied upon (section 9). For this **specific** two-row, single-page, append-only layout, both the observed `Index Scan` and the forced `Seq Scan` return the lower-`ctid` (first-inserted) row first, which is why every drive was **stable at 100/100** within a fixed layout. I therefore state only the **observed** fact - *no application ordering is specified, and the database-selected row here tracks the first-inserted/lower-`ctid` row under the active plan* - and treat the **general** claim, that a different plan, index state, page layout, `VACUUM`/update churn, or concurrency could return the *other* row, as explicitly **`[inferred]`** from the absence of an `ORDER BY` plus the official SQLAlchemy/PostgreSQL semantics (section 9). The forced `Seq Scan` `EXPLAIN` demonstrates the plan can change; it does **not**, on this tiny layout, demonstrate a changed *result row*, and I do not claim it does.

### 7.5 Verdict

Across every disclosed run, the resolved contact - and therefore the alias, owning user, and mailbox derived from it - is determined by **which same-`reply_email` row `.first()` returns**, which under the active plan tracked the contacts' **insertion order**:

| Condition | Insertion order | `ctid=(0,1)` is | Spoof control | Drive result (run 1 / run 2, N=100 each) |
|-----------|-----------------|-----------------|---------------|-------------------------------------------|
| `ab`/`off` | A-then-B | user_A (`id=1`) | **default** | 100/100 E200, `user_id=1`, 100 outbound->`a-dist`, 0 alerts (correct) |
| `ba`/`off` | B-then-A | user_B (`id=1`) | **default** | 100/100 **E214**, `EmailLog=None` (rejected, **no forward**); alerts to user_B: 4 (run1) / 0 (run2, rate-limited) |
| `ab`/`on` | A-then-B | user_A (`id=1`) | `[non-canonical]` | 100/100 E200, `user_id=1`, 100 outbound->`a-dist` (correct) |
| `ba`/`on` | B-then-A | user_B (`id=1`) | `[non-canonical]` | 100/100 E200, **`user_id=2`, 100 outbound->`b-dist` (wrong-user forward)** |

Machine-verified run-to-run stability of the **RESOLUTION** dimension (status + keyed `EmailLog`), by SHA256 over the masked block:

```
ab/off : run1==run2  sha256=7a894de53f06319bb4eea6391a526e68dc136f997708946bc8adc67c90eebd68
ba/off : run1==run2  sha256=d4b7631337f8e015578369789033484d8b7450f9d612d902e2fd488fee3b37f4
ab/on  : run1==run2  sha256=7a894de53f06319bb4eea6391a526e68dc136f997708946bc8adc67c90eebd68   (identical to ab/off - both correct user_A)
ba/on  : run1==run2  sha256=83684ca2ce844b85e6b3274b0706e16259d126fa0e887b628bf6393abda9fb4d
```

- Within each fixed layout the outcome is **deterministic and stable** - 100/100 across two independent fresh-process runs, and the RESOLUTION dimension is byte-identical (SHA256-verified) - it is *not* random per call.
- The resolved **user flips with insertion order** under the identical input (`ab` -> user_A; `ba` -> user_B), which is exactly the ordering-dependence the unordered `.first()` allows.
- Under **default** settings the wrong-user resolution is caught as **E214** (no forward), but still triggers a cross-user `SentAlert` to the wrongly-resolved owner (rate-limited: 4 then 0); the actual wrong-user **forward** (100 outbound under `user_id=2`) is observed only under the **`[non-canonical]`** spoof-disabled fallback.

---
## 8. Edge / secondary conditions

All edge conditions are exercised through the **canonical** `handle_reply()` entry point (or, for `is_reverse_alias`, the labeled `[non-canonical]` predicate), with combined `stdout`+`stderr` captured (no suppression).

### 8.1 E501, E502 (no contact), E502 (inactive user), and the `is_reverse_alias` predicate

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_edge.py 2>&1'
```

**Output (complete, unedited; the machine-verified run-2 stability comparison follows immediately below):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 04:50:19,744 - SL - DEBUG - 18355 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 04:50:24,010 - SL - INFO - 18355 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 04:50:24,012 - SL - INFO - 18355 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 04:50:24,014 - SL - INFO - 18355 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 04:50:24,015 - SL - DEBUG - 18355 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 04:50:24,296 - SL - INFO - 18355 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 04:50:24,312 - SL - DEBUG - 18355 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_word966@sl.local
2026-07-14 04:50:24,319 - SL - INFO - 18355 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 04:50:24,580 - SL - INFO - 18355 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 04:50:24,593 - SL - DEBUG - 18355 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_word271@sl.local
2026-07-14 04:50:24,600 - SL - INFO - 18355 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== (1) E501: reply domain not EMAIL_DOMAIN and not an SLDomain ===
2026-07-14 04:50:24,610 - SL - WARNING - 18355 - "/app/email_handler.py:980" - handle_reply() -  - Reply email reply@notsl.example has wrong domain
rcpt_to='reply@notsl.example' -> delivered=False code='550 SL E501'  (==E501? True)
=== (2) E502: no Contact matches the reply_email ===
2026-07-14 04:50:24,612 - SL - WARNING - 18355 - "/app/email_handler.py:988" - handle_reply() -  - No contact with no-such-contact@sl.local as reverse alias
rcpt_to='no-such-contact@sl.local' -> delivered=False code='550 SL E502 Email not exist'  (==E502? True)
=== (3) E502: contact.user.is_active() is False (soft-deleted user) ===
inactive user delete_on=2026-07-15T04:50:24.604078+00:00  is_active()=False
2026-07-14 04:50:24,616 - SL - WARNING - 18355 - "/app/email_handler.py:991" - handle_reply() -  - User <User 2 Test User edge-inactive@mailbox.test> has been soft deleted
rcpt_to='edge-inactive-reply@sl.local' -> delivered=False code='550 SL E502 Email not exist'  (==E502? True)
=== (4) is_reverse_alias() predicate [app/email_utils.py:1156-1158] ===
is_reverse_alias('edge-active@sl.local')            = True
is_reverse_alias('not-a-reverse-alias@sl.local') = False
DONE
```

- **E501 — wrong reply domain.** `rcpt_to='reply@notsl.example'` does not end with `EMAIL_DOMAIN` and is not an `SLDomain`, so `handle_reply()` logs `Reply email reply@notsl.example has wrong domain` (`email_handler.py:980`) and returns `(False, '550 SL E501')`.
- **E502 — no matching contact.** `rcpt_to='no-such-contact@sl.local'` passes the domain gate but resolves no contact, logging `No contact with no-such-contact@sl.local as reverse alias` (`email_handler.py:988`) and returning `'550 SL E502 Email not exist'`.
- **E502 — inactive (soft-deleted) user.** A contact whose owning user has a future `delete_on` (so `User.is_active()` is `False` — `app/models.py:766-769`) resolves the contact but then fails the active-user gate, logging `User <...> has been soft deleted` (`email_handler.py:991`) and returning `'550 SL E502 Email not exist'`. The printed `delete_on` (a future timestamp) together with `is_active()=False` confirms the precondition; that timestamp is a per-run-varying field (masked in the run-2 comparison below).
- **`is_reverse_alias()` predicate `[non-canonical]`.** `is_reverse_alias('edge-active@sl.local') = True` (a contact exists), `is_reverse_alias('not-a-reverse-alias@sl.local') = False` (no contact and the address does not match the `reply+`/`ra+` suffix clause). This is the two-clause predicate from §6.2.

**Run-2 stability (machine-verified, not asserted).** The identical harness was executed in a **second, fresh OS process**; the two captures (`observe_edge.run1.out`, `observe_edge.run2.out`) were compared after masking only the fields that are *legitimately* per-run-varying — logger timestamps + PID, the randomly-generated alias local-parts, and the computed `delete_on` timestamp in the inactive-user case:

```
$ maskdoc() { sed -E \
    "s/^20[0-9]{2}-[0-9]{2}-[0-9]{2} [0-9:,]+ - SL - [A-Z]+ - [0-9]+ - /<TS> - SL - LEVEL - <PID> - /; \
     s/[a-z]+_[a-z]+[0-9]+@sl\.local/<RANDOM_ALIAS>@sl.local/g; \
     s/sl\.[a-z0-9]+\.[a-z0-9]+@sl\.local/<RANDOM_REVERSE_ALIAS>@sl.local/g; \
     s/<[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+@sl\.local>/<SL_MESSAGE_ID>/g; \
     s/20[0-9]{2}-[0-9]{2}-[0-9]{2}T[0-9:]+\.[0-9]+\+00:00/<DELETE_ON>/g" "$1"; }
$ diff <(maskdoc observe_edge.run1.out) <(maskdoc observe_edge.run2.out); echo "exit=$?"
exit=0
$ sha256sum <(maskdoc observe_edge.run1.out) <(maskdoc observe_edge.run2.out)
d12b8221075526b4b92910789fd4166149c5baaa75e19ed359c7487d63bea35d  /dev/fd/63
d12b8221075526b4b92910789fd4166149c5baaa75e19ed359c7487d63bea35d  /dev/fd/62
```

`diff` produced **no output (exit 0)** and both masked runs share one SHA256, so every semantic value (status codes, resolved identities, and persisted state shown above) is **byte-identical across two fresh processes**; only the enumerated timing/random fields differ.

### 8.2 Normalization semantics — inbound spelling aliasing vs. exact-equality lookup (corrected)

This experiment establishes the **precise** normalization semantics and corrects the earlier conflation: `normalize_reply_email()` transforms only the **inbound** `rcpt_to`; the stored `Contact.reply_email` is compared by **exact equality**. Multiple inbound spellings therefore collapse onto **one** stored key, but a **multi-row** match still requires **more than one stored row** holding that exact key.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_norm.py 2>&1'
```

**Output (complete, unedited; the machine-verified run-2 stability comparison follows immediately below):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 21:38:56,672 - SL - DEBUG - 9160 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 21:39:01,119 - SL - INFO - 9160 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-13 21:39:01,122 - SL - INFO - 9160 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-13 21:39:01,124 - SL - INFO - 9160 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-13 21:39:01,125 - SL - DEBUG - 9160 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 21:39:01,409 - SL - INFO - 9160 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 21:39:01,427 - SL - DEBUG - 9160 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_word312@sl.local
2026-07-13 21:39:01,435 - SL - INFO - 9160 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== stored Contact.reply_email = 'norm_key@sl.local' (single row) ===
--- normalize_reply_email() maps each INBOUND spelling onto the stored key ---
  inbound='norm_key@sl.local'              -> normalize_reply_email -> 'norm_key@sl.local'  (==stored? True)
  inbound='norm key@sl.local'              -> normalize_reply_email -> 'norm_key@sl.local'  (==stored? True)
  inbound='norm#key@sl.local'              -> normalize_reply_email -> 'norm_key@sl.local'  (==stored? True)
  inbound='norm~key@sl.local'              -> normalize_reply_email -> 'norm_key@sl.local'  (==stored? True)
--- exact-equality proof: query uses normalized INBOUND vs raw STORED ---
  Contact.get_by(reply_email='norm key@sl.local')            -> None
  Contact.get_by(reply_email=normalize('norm key@sl.local')) -> <Contact 1 ext@nowhere.net 2>
--- canonical handle_reply() with each inbound spelling resolves the ONE stored contact ---
2026-07-13 21:39:01,447 - SL - INFO - 9160 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 21:39:01,453 - SL - DEBUG - 9160 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 ext@nowhere.net 2>, <User 1 Test User norm-user@mailbox.test>, <Mailbox 1 norm-user@mailbox.test>
2026-07-13 21:39:01,459 - SL - DEBUG - 9160 - "/app/email_handler.py:1171" - handle_reply() -  - From header is word_word312@sl.local
2026-07-13 21:39:01,460 - SL - DEBUG - 9160 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: norm_key@sl.local, new: Ext <ext@nowhere.net>
2026-07-13 21:39:01,460 - SL - DEBUG - 9160 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 21:39:01,462 - SL - DEBUG - 9160 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178397874146.9160.6600060960573048428.1@sl.local>
2026-07-13 21:39:01,469 - SL - DEBUG - 9160 - "/app/email_handler.py:1212" - handle_reply() -  - send email from word_word312@sl.local to ext@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 21:39:01,473 - SL - DEBUG - 9160 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'norm 0', from 'word_word312@sl.local' to 'Ext <ext@nowhere.net>'
  inbound='norm_key@sl.local'              -> delivered=True code='250 Message accepted for delivery' EmailLog.contact_id=1
2026-07-13 21:39:01,482 - SL - INFO - 9160 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 21:39:01,484 - SL - DEBUG - 9160 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 2> for <Contact 1 ext@nowhere.net 2>, <User 1 Test User norm-user@mailbox.test>, <Mailbox 1 norm-user@mailbox.test>
2026-07-13 21:39:01,489 - SL - DEBUG - 9160 - "/app/email_handler.py:1171" - handle_reply() -  - From header is word_word312@sl.local
2026-07-13 21:39:01,490 - SL - DEBUG - 9160 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: norm_key@sl.local, new: Ext <ext@nowhere.net>
2026-07-13 21:39:01,490 - SL - DEBUG - 9160 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 21:39:01,492 - SL - DEBUG - 9160 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178397874149.9160.14513197473063157976.2@sl.local>
2026-07-13 21:39:01,498 - SL - DEBUG - 9160 - "/app/email_handler.py:1212" - handle_reply() -  - send email from word_word312@sl.local to ext@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 21:39:01,500 - SL - DEBUG - 9160 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'norm 1', from 'word_word312@sl.local' to 'Ext <ext@nowhere.net>'
  inbound='norm key@sl.local'              -> delivered=True code='250 Message accepted for delivery' EmailLog.contact_id=1
2026-07-13 21:39:01,508 - SL - INFO - 9160 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 21:39:01,511 - SL - DEBUG - 9160 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 3> for <Contact 1 ext@nowhere.net 2>, <User 1 Test User norm-user@mailbox.test>, <Mailbox 1 norm-user@mailbox.test>
2026-07-13 21:39:01,516 - SL - DEBUG - 9160 - "/app/email_handler.py:1171" - handle_reply() -  - From header is word_word312@sl.local
2026-07-13 21:39:01,517 - SL - DEBUG - 9160 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: norm_key@sl.local, new: Ext <ext@nowhere.net>
2026-07-13 21:39:01,517 - SL - DEBUG - 9160 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 21:39:01,519 - SL - DEBUG - 9160 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178397874151.9160.10709628307354080399.3@sl.local>
2026-07-13 21:39:01,524 - SL - DEBUG - 9160 - "/app/email_handler.py:1212" - handle_reply() -  - send email from word_word312@sl.local to ext@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 21:39:01,527 - SL - DEBUG - 9160 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'norm 2', from 'word_word312@sl.local' to 'Ext <ext@nowhere.net>'
  inbound='norm#key@sl.local'              -> delivered=True code='250 Message accepted for delivery' EmailLog.contact_id=1
2026-07-13 21:39:01,535 - SL - INFO - 9160 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 21:39:01,537 - SL - DEBUG - 9160 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 4> for <Contact 1 ext@nowhere.net 2>, <User 1 Test User norm-user@mailbox.test>, <Mailbox 1 norm-user@mailbox.test>
2026-07-13 21:39:01,542 - SL - DEBUG - 9160 - "/app/email_handler.py:1171" - handle_reply() -  - From header is word_word312@sl.local
2026-07-13 21:39:01,543 - SL - DEBUG - 9160 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: norm_key@sl.local, new: Ext <ext@nowhere.net>
2026-07-13 21:39:01,544 - SL - DEBUG - 9160 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 21:39:01,545 - SL - DEBUG - 9160 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178397874154.9160.15015114200142604317.4@sl.local>
2026-07-13 21:39:01,551 - SL - DEBUG - 9160 - "/app/email_handler.py:1212" - handle_reply() -  - send email from word_word312@sl.local to ext@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 21:39:01,554 - SL - DEBUG - 9160 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'norm 3', from 'word_word312@sl.local' to 'Ext <ext@nowhere.net>'
  inbound='norm~key@sl.local'              -> delivered=True code='250 Message accepted for delivery' EmailLog.contact_id=1
--- rows sharing the stored key = 1 (multi-row match needs >1 STORED row with same key) ---
DONE
```

- **Inbound many-to-one.** A single stored `Contact.reply_email='norm_key@sl.local'`. Four distinct inbound spellings — `'norm_key@sl.local'`, `'norm key@sl.local'` (space), `'norm#key@sl.local'`, `'norm~key@sl.local'` — all normalize to `'norm_key@sl.local'` (`==stored? True`), because the disallowed characters (space, `#`, `~`) are each replaced by `_` (`app/email_validation.py:33`, using `_ALLOWED_CHARS` at `:9`).
- **Exact-equality lookup (the correction).** `Contact.get_by(reply_email='norm key@sl.local') -> None` (the **raw** inbound spelling is *not* matched, because the query compares against the raw **stored** value), whereas `Contact.get_by(reply_email=normalize('norm key@sl.local')) -> <Contact 1>`. So the match happens **after** the caller normalizes the inbound value — the lookup itself does not normalize stored rows.
- **Consequence for multi-row.** All four inbound spellings drive canonical `handle_reply()` to E200 resolving the same `contact_id=1`; and `rows sharing the stored key = 1`. A multi-row (hence order-dependent) match therefore requires **more than one stored row** with the same normalized key — exactly the duplicate condition seeded in §4.2/§7 — not merely different inbound spellings of one stored value.

**Run-2 stability (machine-verified, not asserted).** The identical harness was executed in a **second, fresh OS process**; the two captures (`observe_norm.run1.out`, `observe_norm.run2.out`) were compared after masking only the fields that are *legitimately* per-run-varying — logger timestamps + PID, the randomly-generated alias local-parts, and the `sl_message_id` tokens:

```
$ maskdoc() { sed -E \
    "s/^20[0-9]{2}-[0-9]{2}-[0-9]{2} [0-9:,]+ - SL - [A-Z]+ - [0-9]+ - /<TS> - SL - LEVEL - <PID> - /; \
     s/[a-z]+_[a-z]+[0-9]+@sl\.local/<RANDOM_ALIAS>@sl.local/g; \
     s/sl\.[a-z0-9]+\.[a-z0-9]+@sl\.local/<RANDOM_REVERSE_ALIAS>@sl.local/g; \
     s/<[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+@sl\.local>/<SL_MESSAGE_ID>/g; \
     s/20[0-9]{2}-[0-9]{2}-[0-9]{2}T[0-9:]+\.[0-9]+\+00:00/<DELETE_ON>/g" "$1"; }
$ diff <(maskdoc observe_norm.run1.out) <(maskdoc observe_norm.run2.out); echo "exit=$?"
exit=0
$ sha256sum <(maskdoc observe_norm.run1.out) <(maskdoc observe_norm.run2.out)
8c9b1f1fd1bcc115ccac0168c332ff9af3d76fbb29af335c2cab18c3a501d765  /dev/fd/63
8c9b1f1fd1bcc115ccac0168c332ff9af3d76fbb29af335c2cab18c3a501d765  /dev/fd/62
```

`diff` produced **no output (exit 0)** and both masked runs share one SHA256, so every semantic value (status codes, resolved identities, and persisted state shown above) is **byte-identical across two fresh processes**; only the enumerated timing/random fields differ.

### 8.3 Second lookup site — `replace_header_when_reply()` (`email_handler.py:364`)

During the reply, `handle_reply()` calls `replace_header_when_reply()` for the `TO` header (`email_handler.py:1179`) and the `CC` header (`email_handler.py:1181`); that function performs a **second** `Contact.get_by(reply_email=...)` (`email_handler.py:364`) to restore each reverse-alias in the header back to the original contact address (or raises `NonReverseAliasInReplyPhase` if none matches). This is a second site subject to the same non-unique `.first()` semantics.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_second.py 2>&1'
```

**Output (complete, unedited; the machine-verified run-2 stability comparison follows immediately below):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 04:51:48,291 - SL - DEBUG - 18440 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 04:51:52,495 - SL - INFO - 18440 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 04:51:52,498 - SL - INFO - 18440 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 04:51:52,499 - SL - INFO - 18440 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 04:51:52,500 - SL - DEBUG - 18440 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 04:51:52,783 - SL - INFO - 18440 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 04:51:52,800 - SL - DEBUG - 18440 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_word452@sl.local
2026-07-14 04:51:52,807 - SL - INFO - 18440 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== second lookup site: replace_header_when_reply() [email_handler.py:364] via TO & CC ===
primary rcpt_to='second-r1@sl.local' (resolved at :986); TO header='second-r1@sl.local' -> contact c1.id=1 website='w1@nowhere.net'
CC header='second-r2@sl.local' -> contact c2.id=2 website='w2@nowhere.net'
2026-07-14 04:51:52,821 - SL - INFO - 18440 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 04:51:52,826 - SL - DEBUG - 18440 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 w1@nowhere.net 2>, <User 1 Test User second-user@mailbox.test>, <Mailbox 1 second-user@mailbox.test>
2026-07-14 04:51:52,832 - SL - DEBUG - 18440 - "/app/email_handler.py:1171" - handle_reply() -  - From header is list_word452@sl.local
2026-07-14 04:51:52,834 - SL - DEBUG - 18440 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: second-r1@sl.local, new: W1 <w1@nowhere.net>
2026-07-14 04:51:52,835 - SL - DEBUG - 18440 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace Cc header, old: second-r2@sl.local, new: W2 <w2@nowhere.net>
2026-07-14 04:51:52,837 - SL - DEBUG - 18440 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178400471283.18440.9064413032140446819.1@sl.local>
2026-07-14 04:51:52,844 - SL - DEBUG - 18440 - "/app/email_handler.py:1212" - handle_reply() -  - send email from list_word452@sl.local to w1@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-14 04:51:52,847 - SL - DEBUG - 18440 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'second-lookup', from 'list_word452@sl.local' to 'W1 <w1@nowhere.net>'
handle_reply -> delivered=True code='250 Message accepted for delivery' (==E200? True)
EmailLog id=1 contact_id=1 alias_id=2 user_id=1 is_reply=True
rewritten msg[To] = 'W1 <w1@nowhere.net>'
rewritten msg[Cc] = 'W2 <w2@nowhere.net>'
DONE
```

- The canonical `handle_reply()` call carries a `TO` header `second-r1@sl.local` (resolving contact `c1.id=1`, `w1@nowhere.net`) and a `CC` header `second-r2@sl.local` (resolving contact `c2.id=2`, `w2@nowhere.net`). The second-site lookup rewrites both: `Replace To header, old: second-r1@sl.local, new: W1 <w1@nowhere.net>` and `Replace Cc header, old: second-r2@sl.local, new: W2 <w2@nowhere.net>` (`email_handler.py:380`).
- Result: `delivered=True code='250 Message accepted for delivery'` (E200), `EmailLog id=1 contact_id=1 alias_id=2 user_id=1 is_reply=True`, and the rewritten `msg[To]='W1 <w1@nowhere.net>'`, `msg[Cc]='W2 <w2@nowhere.net>'`. Two **distinct** contacts are resolved through the second site in one reply, confirming the site is live on the canonical path.

**Run-2 stability (machine-verified, not asserted).** The identical harness was executed in a **second, fresh OS process**; the two captures (`observe_second.run1.out`, `observe_second.run2.out`) were compared after masking only the fields that are *legitimately* per-run-varying — logger timestamps + PID and the randomly-generated alias local-parts:

```
$ maskdoc() { sed -E \
    "s/^20[0-9]{2}-[0-9]{2}-[0-9]{2} [0-9:,]+ - SL - [A-Z]+ - [0-9]+ - /<TS> - SL - LEVEL - <PID> - /; \
     s/[a-z]+_[a-z]+[0-9]+@sl\.local/<RANDOM_ALIAS>@sl.local/g; \
     s/sl\.[a-z0-9]+\.[a-z0-9]+@sl\.local/<RANDOM_REVERSE_ALIAS>@sl.local/g; \
     s/<[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+@sl\.local>/<SL_MESSAGE_ID>/g; \
     s/20[0-9]{2}-[0-9]{2}-[0-9]{2}T[0-9:]+\.[0-9]+\+00:00/<DELETE_ON>/g" "$1"; }
$ diff <(maskdoc observe_second.run1.out) <(maskdoc observe_second.run2.out); echo "exit=$?"
exit=0
$ sha256sum <(maskdoc observe_second.run1.out) <(maskdoc observe_second.run2.out)
876f0d4d7382500426c620413ab1b7cdf5af3a1951d8ca3b74121ce5518dd910  /dev/fd/63
876f0d4d7382500426c620413ab1b7cdf5af3a1951d8ca3b74121ce5518dd910  /dev/fd/62
```

`diff` produced **no output (exit 0)** and both masked runs share one SHA256, so every semantic value (status codes, resolved identities, and persisted state shown above) is **byte-identical across two fresh processes**; only the enumerated timing/random fields differ.

### 8.4 Five-way E214 sender-authorization matrix — who a reply reaches vs. who is rejected

§8.1–§8.3 covered the domain gate (E501), the no-contact gate (E502), and the second lookup site. This subsection exercises the **authorization** branch that sits *after* the contact is resolved: once `Contact.get_by(reply_email=...)` (`email_handler.py:986`) has resolved a contact — and therefore `alias = contact.alias` (`email_handler.py:994`) and `user = alias.user` (`email_handler.py:1004`) — the handler must decide whether the message's `mail_from` is *authorized* to send through that alias. That decision is made by `get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:1364`, invoked at `email_handler.py:1019`), which matches `mail_from` against `alias.mailboxes[].email` **and** each `mailbox.authorized_addresses[].email`. On no match, and when spoofing-check is not disabled, the handler calls `handle_unknown_mailbox(...)` (`email_handler.py:1390`) which returns **E214** (`email_handler.py:1034`) and sends an alert to `user.email` — i.e. the **resolved alias owner**, `email_handler.py:1393` — *not* the sender.

The seed (Section A of the capture) creates three users, one default alias each, and **two** `Contact` rows that deliberately share `reply_email='f07-shared@sl.local'` on aliases owned by **different** users (`ctid=(0,1)` on aliasA/user_A and `ctid=(0,2)` on aliasB/user_B) — permitted because there is no `UNIQUE` constraint on `reply_email` (see §5, §6). `Contact.get_by(reply_email=SHARED).first()` resolves to `id=1 alias_id=1 user_id=1` (aliasA/user_A). Holding that resolved owner fixed, Section B drives the canonical `handle_reply()` five times, varying only the envelope `mail_from` across five sender classes.

The complete unedited capture below spans **all** of the harness's Sections A–E; it is shown once, here, in full. §8.4 analyzes Section A (seed) and Section B (the E214 matrix); §8.5 analyzes Section C (E504); §8.6 analyzes Section D (adversarial `is_reverse_alias()`); §8.7 analyzes Section E (adversarial `replace_header_when_reply()`). §8.5–§8.7 reference slices of this same capture rather than presenting separately-edited excerpts.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_f07.py 2>&1'
```

**Output (complete, unedited; run 1 of 2 — see §8.8 for the machine-verified stability comparison across both runs):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 03:21:51,464 - SL - DEBUG - 16950 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 03:21:55,846 - SL - INFO - 16950 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 03:21:55,849 - SL - INFO - 16950 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 03:21:55,850 - SL - INFO - 16950 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 03:21:55,852 - SL - DEBUG - 16950 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 03:21:56,133 - SL - INFO - 16950 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 03:21:56,394 - SL - INFO - 16950 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 03:21:56,653 - SL - INFO - 16950 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
========== SECTION A: SEED ==========
EMAIL_DOMAIN='sl.local'  NOT_SEND_EMAIL=True  SHARED='f07-shared@sl.local'
user_a.id=1 email='usera@mailbox.test' alias_a.id=1 alias_a.email='simplelogin-newsletter.list834@sl.local'
user_b.id=2 email='userb@mailbox.test' alias_b.id=2 alias_b.email='simplelogin-newsletter.word454@sl.local'
user_c.id=3 email='userc@mailbox.test' alias_c.id=3 alias_c.email='simplelogin-newsletter.list314@sl.local'
contacts sharing reply_email='f07-shared@sl.local': count=2
  ctid=(0,1) id=1 alias_id=1 user_id=1 website='ca@nowhere.net'
  ctid=(0,2) id=2 alias_id=2 user_id=2 website='cb@nowhere.net'
Contact.get_by(reply_email=SHARED).first() -> id=1 alias_id=1 user_id=1 (resolved owner = user_a? True)

========== SECTION B: FIVE-WAY E214 SENDER MATRIX (rcpt_to resolves aliasA/user_A) ==========
--- [B1 resolved-owner] sender_class=resolved_owner ---
    envelope.mail_from = 'usera@mailbox.test'   rcpt_to = 'f07-shared@sl.local'   Message-ID = '<f07-b1@sl.local>'
2026-07-14 03:21:56,705 - SL - INFO - 16950 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 03:21:56,711 - SL - DEBUG - 16950 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 ca@nowhere.net 1>, <User 1 Test User usera@mailbox.test>, <Mailbox 1 usera@mailbox.test>
2026-07-14 03:21:56,716 - SL - DEBUG - 16950 - "/app/email_handler.py:1171" - handle_reply() -  - From header is simplelogin-newsletter.list834@sl.local
2026-07-14 03:21:56,716 - SL - DEBUG - 16950 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the To header. Old value simplelogin-newsletter.list834@sl.local
2026-07-14 03:21:56,716 - SL - DEBUG - 16950 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-14 03:21:56,718 - SL - DEBUG - 16950 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178399931671.16950.14192506373636101697.1@sl.local>
2026-07-14 03:21:56,723 - SL - WARNING - 16950 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-14 03:21:56,725 - SL - DEBUG - 16950 - "/app/email_handler.py:1212" - handle_reply() -  - send email from simplelogin-newsletter.list834@sl.local to ca@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-14 03:21:56,729 - SL - DEBUG - 16950 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply subject', from 'simplelogin-newsletter.list834@sl.local' to 'None'
    RESULT delivered=True code='250 Message accepted for delivery'  (== status.E200)
    EmailLog: id=1 contact_id=1 alias_id=1 user_id=1 mailbox_id=1 is_reply=True
    outbound stored SendRequest count = 1
      -> envelope_to='ca@nowhere.net' subject='reply subject' msg[From]='simplelogin-newsletter.list834@sl.local'
    NEW SentAlert rows during call = 0
    DB deltas: EmailLog 0->1  SentAlert 0->0
    handle_reply() wall-clock = 27.0 ms

--- [B2 other-dup-owner] sender_class=other_dup_owner ---
    envelope.mail_from = 'userb@mailbox.test'   rcpt_to = 'f07-shared@sl.local'   Message-ID = '<f07-b2@sl.local>'
2026-07-14 03:21:56,744 - SL - INFO - 16950 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 03:21:56,746 - SL - WARNING - 16950 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: userb@mailbox.test. msg from header: external-contact@nowhere.net, reverse-alias f07-shared@sl.local, <Alias 1 simplelogin-newsletter.list834@sl.local> <User 1 Test User usera@mailbox.test> <Contact 1 ca@nowhere.net 1>
2026-07-14 03:21:56,765 - SL - DEBUG - 16950 - "/app/app/email_utils.py:303" - send_email() -  - send email to usera@mailbox.test, subject 'Attempt to use your alias simplelogin-newsletter.list834@sl.local from userb@mailbox.test'
2026-07-14 03:21:56,772 - SL - DEBUG - 16950 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Attempt to use your alias simplelogin-newsletter.list834@sl.local from userb@mailbox.test', from '"noreply@sl.local" <noreply@sl.local>' to 'usera@mailbox.test'
    RESULT delivered=False code='250 SL E214 Unauthorized for using reverse alias'  (== status.E214)
    EmailLog: None (no log persisted on this path)
    outbound stored SendRequest count = 1
      -> envelope_to='usera@mailbox.test' subject='Attempt to use your alias simplelogin-newsletter.list834@sl.local from userb@mailbox.test' msg[From]='"noreply@sl.local" <noreply@sl.local>'
    NEW SentAlert rows during call = 1
    DB deltas: EmailLog 1->1  SentAlert 0->1
    handle_reply() wall-clock = 30.8 ms

--- [B3 unrelated-verified] sender_class=unrelated_verified ---
    envelope.mail_from = 'userc@mailbox.test'   rcpt_to = 'f07-shared@sl.local'   Message-ID = '<f07-b3@sl.local>'
2026-07-14 03:21:56,788 - SL - INFO - 16950 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 03:21:56,789 - SL - WARNING - 16950 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: userc@mailbox.test. msg from header: external-contact@nowhere.net, reverse-alias f07-shared@sl.local, <Alias 1 simplelogin-newsletter.list834@sl.local> <User 1 Test User usera@mailbox.test> <Contact 1 ca@nowhere.net 1>
2026-07-14 03:21:56,806 - SL - DEBUG - 16950 - "/app/app/email_utils.py:303" - send_email() -  - send email to usera@mailbox.test, subject 'Attempt to use your alias simplelogin-newsletter.list834@sl.local from userc@mailbox.test'
2026-07-14 03:21:56,812 - SL - DEBUG - 16950 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Attempt to use your alias simplelogin-newsletter.list834@sl.local from userc@mailbox.test', from '"noreply@sl.local" <noreply@sl.local>' to 'usera@mailbox.test'
    RESULT delivered=False code='250 SL E214 Unauthorized for using reverse alias'  (== status.E214)
    EmailLog: None (no log persisted on this path)
    outbound stored SendRequest count = 1
      -> envelope_to='usera@mailbox.test' subject='Attempt to use your alias simplelogin-newsletter.list834@sl.local from userc@mailbox.test' msg[From]='"noreply@sl.local" <noreply@sl.local>'
    NEW SentAlert rows during call = 1
    DB deltas: EmailLog 1->1  SentAlert 1->2
    handle_reply() wall-clock = 27.4 ms

--- [B4 unknown-sender] sender_class=unknown_sender ---
    envelope.mail_from = 'stranger@nowhere.test'   rcpt_to = 'f07-shared@sl.local'   Message-ID = '<f07-b4@sl.local>'
2026-07-14 03:21:56,825 - SL - INFO - 16950 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 03:21:56,827 - SL - WARNING - 16950 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: stranger@nowhere.test. msg from header: external-contact@nowhere.net, reverse-alias f07-shared@sl.local, <Alias 1 simplelogin-newsletter.list834@sl.local> <User 1 Test User usera@mailbox.test> <Contact 1 ca@nowhere.net 1>
2026-07-14 03:21:56,843 - SL - DEBUG - 16950 - "/app/app/email_utils.py:303" - send_email() -  - send email to usera@mailbox.test, subject 'Attempt to use your alias simplelogin-newsletter.list834@sl.local from stranger@nowhere.test'
2026-07-14 03:21:56,850 - SL - DEBUG - 16950 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Attempt to use your alias simplelogin-newsletter.list834@sl.local from stranger@nowhere.test', from '"noreply@sl.local" <noreply@sl.local>' to 'usera@mailbox.test'
    RESULT delivered=False code='250 SL E214 Unauthorized for using reverse alias'  (== status.E214)
    EmailLog: None (no log persisted on this path)
    outbound stored SendRequest count = 1
      -> envelope_to='usera@mailbox.test' subject='Attempt to use your alias simplelogin-newsletter.list834@sl.local from stranger@nowhere.test' msg[From]='"noreply@sl.local" <noreply@sl.local>'
    NEW SentAlert rows during call = 1
    DB deltas: EmailLog 1->1  SentAlert 2->3
    handle_reply() wall-clock = 27.6 ms

--- [B5 malformed-sender] sender_class=malformed_sender ---
    envelope.mail_from = 'not-an-email'   rcpt_to = 'f07-shared@sl.local'   Message-ID = '<f07-b5@sl.local>'
2026-07-14 03:21:56,863 - SL - INFO - 16950 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 03:21:56,865 - SL - WARNING - 16950 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: not-an-email. msg from header: external-contact@nowhere.net, reverse-alias f07-shared@sl.local, <Alias 1 simplelogin-newsletter.list834@sl.local> <User 1 Test User usera@mailbox.test> <Contact 1 ca@nowhere.net 1>
2026-07-14 03:21:56,883 - SL - DEBUG - 16950 - "/app/app/email_utils.py:303" - send_email() -  - send email to usera@mailbox.test, subject 'Attempt to use your alias simplelogin-newsletter.list834@sl.local from not-an-email'
2026-07-14 03:21:56,897 - SL - DEBUG - 16950 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Attempt to use your alias simplelogin-newsletter.list834@sl.local from not-an-email', from '"noreply@sl.local" <noreply@sl.local>' to 'usera@mailbox.test'
    RESULT delivered=False code='250 SL E214 Unauthorized for using reverse alias'  (== status.E214)
    EmailLog: None (no log persisted on this path)
    outbound stored SendRequest count = 1
      -> envelope_to='usera@mailbox.test' subject='Attempt to use your alias simplelogin-newsletter.list834@sl.local from not-an-email' msg[From]='"noreply@sl.local" <noreply@sl.local>'
    NEW SentAlert rows during call = 1
    DB deltas: EmailLog 1->1  SentAlert 3->4
    handle_reply() wall-clock = 37.6 ms

========== SECTION C: REACHABLE E504 (resolved alias owner cannot send/receive) ==========
--- C1: user_A.disabled=True (delete_on=None => is_active()=True, can_send_or_receive()=False) ---
2026-07-14 03:21:56,908 - SL - INFO - 16950 - "/app/app/models.py:888" - can_send_or_receive() -  - User <User 1 Test User usera@mailbox.test> is disabled. Cannot receive or send emails
    user_a.disabled=True delete_on=None is_active()=True can_send_or_receive()=False
--- [C1 disabled-user] sender_class=resolved_owner_disabled ---
    envelope.mail_from = 'usera@mailbox.test'   rcpt_to = 'f07-shared@sl.local'   Message-ID = '<f07-c1@sl.local>'
2026-07-14 03:21:56,915 - SL - INFO - 16950 - "/app/app/models.py:888" - can_send_or_receive() -  - User <User 1 Test User usera@mailbox.test> is disabled. Cannot receive or send emails
2026-07-14 03:21:56,915 - SL - INFO - 16950 - "/app/email_handler.py:1008" - handle_reply() -  - User <User 1 Test User usera@mailbox.test> cannot send emails
    RESULT delivered=False code='550 SL E504 Account disabled'  (== status.E504)
    EmailLog: None (no log persisted on this path)
    outbound stored SendRequest count = 0
    NEW SentAlert rows during call = 0
    DB deltas: EmailLog 1->1  SentAlert 4->4
    handle_reply() wall-clock = 1.9 ms

--- C2: user_A.disabled=False, delete_on=now (scheduled deletion) ---
2026-07-14 03:21:56,928 - SL - INFO - 16950 - "/app/app/models.py:891" - can_send_or_receive() -  - User <User 1 Test User usera@mailbox.test> is scheduled to be deleted. Cannot receive or send emails
    user_a.disabled=False delete_on set=True is_active()=True can_send_or_receive()=False
--- [C2 delete_on-user] sender_class=resolved_owner_delete_on ---
    envelope.mail_from = 'usera@mailbox.test'   rcpt_to = 'f07-shared@sl.local'   Message-ID = '<f07-c2@sl.local>'
2026-07-14 03:21:56,936 - SL - INFO - 16950 - "/app/app/models.py:891" - can_send_or_receive() -  - User <User 1 Test User usera@mailbox.test> is scheduled to be deleted. Cannot receive or send emails
2026-07-14 03:21:56,937 - SL - INFO - 16950 - "/app/email_handler.py:1008" - handle_reply() -  - User <User 1 Test User usera@mailbox.test> cannot send emails
    RESULT delivered=False code='550 SL E504 Account disabled'  (== status.E504)
    EmailLog: None (no log persisted on this path)
    outbound stored SendRequest count = 0
    NEW SentAlert rows during call = 0
    DB deltas: EmailLog 1->1  SentAlert 4->4
    handle_reply() wall-clock = 2.0 ms

--- C3: RESTORE user_A (disabled=False, delete_on=None) ---
    RESTORED user_a.disabled=False delete_on=None is_active()=True can_send_or_receive()=True

========== SECTION D: ADVERSARIAL is_reverse_alias() PREDICATE ==========
  is_reverse_alias(empty='') = False  (returned a bool; no injection)
  is_reverse_alias(sql_injection="' OR '1'='1' --@sl.local") = False  (returned a bool; no injection)
  is_reverse_alias(sql_drop="x'; DROP TABLE contact;--@sl.local") = False  (returned a bool; no injection)
  is_reverse_alias(crlf_injection='a@sl.local\r\nBcc: attacker@evil.test') = False  (returned a bool; no injection)
  is_reverse_alias(unicode='réply中文@sl.local') = False  (returned a bool; no injection)
  is_reverse_alias(overlong_5000='aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...(5009 chars)') = False  (returned a bool; no injection)
  is_reverse_alias(null_byte='a\x00b@sl.local') RAISED ValueError: 'A string literal cannot contain NUL (0x00) characters.'  (safe: driver rejected input; no injection/execution)
  Contact rowcount before=2 after=2  (unchanged: predicate performs no writes)

========== SECTION E: ADVERSARIAL replace_header_when_reply() (second lookup site) ==========
  E1 To (old) = 'Alias <simplelogin-newsletter.list834@sl.local>, RA <f07-shared@sl.local>'
2026-07-14 03:21:56,962 - SL - DEBUG - 16950 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: Alias <simplelogin-newsletter.list834@sl.local>, RA <f07-shared@sl.local>, new: CA <ca@nowhere.net>
  E1 To (new) = 'CA <ca@nowhere.net>'  (reverse-alias replaced by contact website_email; alias skipped)
  E2 modern EmailMessage policy REJECTS CRLF at set-time: ValueError 'Header values may not contain linefeed or carriage return characters' (defense-in-depth: the pipeline message object cannot hold an injected header)
  E3 compat32 To (raw) = ['first@evil.test,\r\n\tsecond@evil.test']  has_CR=True has_LF=True
2026-07-14 03:21:56,963 - SL - WARNING - 16950 - "/app/email_handler.py:366" - replace_header_when_reply() -  - email first@evil.test contained in To header in reply phase must be reply emails. headers:['first@evil.test,\tsecond@evil.test']
  E3 raised NonReverseAliasInReplyPhase (non-reverse addr) - CAUGHT; the function got PAST the \r/\n strip to address parsing, so no CRLF survives into any rewritten header
  E4 compat32 To (adversarial) = ['"x"; DROP TABLE contact;--���@evil.test']
2026-07-14 03:21:56,964 - SL - WARNING - 16950 - "/app/email_handler.py:366" - replace_header_when_reply() -  - email  contained in To header in reply phase must be reply emails. headers:['"x"; DROP TABLE contact;--���@evil.test']
  E4 raised NonReverseAliasInReplyPhase - CAUGHT (safe; no crash, no write)
  Contact rowcount before=2 after=2  (unchanged: header rewrite performs no Contact writes)

DONE
```

**Section B matrix (resolved owner fixed = aliasA / user_A; only `mail_from` varies):**

| # | Sender class — `mail_from` | Authorized for aliasA? | Result code | `EmailLog` | Alert recipient | `SentAlert` Δ |
|---|---|---|---|---|---|---|
| B1 | resolved-owner `usera@mailbox.test` | yes (owns aliasA's mailbox) | **E200** `250 Message accepted for delivery` | `id=1 contact_id=1 alias_id=1 user_id=1 mailbox_id=1` | — (delivered to contact `ca@nowhere.net`) | 0→0 |
| B2 | other-dup-owner `userb@mailbox.test` | no | **E214** `250 SL E214 Unauthorized...` | `None` | `usera@mailbox.test` | 0→1 |
| B3 | unrelated-verified `userc@mailbox.test` | no | **E214** | `None` | `usera@mailbox.test` | 1→2 |
| B4 | unknown-sender `stranger@nowhere.test` | no | **E214** | `None` | `usera@mailbox.test` | 2→3 |
| B5 | malformed-sender `not-an-email` | no | **E214** | `None` | `usera@mailbox.test` | 3→4 |

Cause → effect:

- **Only B1 delivers.** The resolved owner's own mailbox address matches `get_mailbox_from_mail_from()`, so the message is accepted (E200), an `EmailLog` is persisted (`contact_id=1 alias_id=1 user_id=1`, `email_handler.py:1051`), and the outbound `SendRequest` is addressed to the real contact `ca@nowhere.net` with the `From` rewritten to the alias `simplelogin-newsletter.list834@sl.local`.
- **B2–B5 are all rejected with E214**, no `EmailLog` is persisted, and each raises exactly one `SentAlert`. The rejection is identical whether the sender is a *legitimate but different SimpleLogin user* (B2 `userb@` — who actually owns the **other** duplicate contact `cb@nowhere.net`; B3 `userc@`), an unknown external address (B4), or a syntactically malformed one (B5).
- **The disclosure is the point.** In every E214 case the alert — subject `Attempt to use your alias simplelogin-newsletter.list834@sl.local from <mail_from>` — is delivered to `usera@mailbox.test`, the **resolved** alias owner (`handle_unknown_mailbox()` → `send_email(... user.email ...)`, logged at `app/email_utils.py:303`), regardless of who actually sent. When the shared `reply_email` resolves to the *wrong* owner (the §10 wrong-user condition), the E214 alert — including the alias address and the sender's address — is disclosed to that wrong owner. This is the alerting-path consequence of the same non-unique `.first()` resolution and is analyzed further as a privacy consequence in the security matrix (F-04, §14).
- **`SentAlert` climbs 0→4 across B2–B5 in a single fresh run.** `send_email_with_rate_control()` caps alerts at 4 per recipient per alert-type per 24h; the four E214 cases here land exactly at that cap. `EmailLog` stays at 1 throughout B2–B5 (the E214 path persists no log), confirming the before/after DB state for each drive.

### 8.5 Reachable E504 — resolved alias owner cannot send or receive

E504 (`550 SL E504 Account disabled`) is returned at `email_handler.py:1008` when `user.can_send_or_receive()` is false for the **resolved** alias owner. `can_send_or_receive()` (`app/models.py:886`) returns false in two independent ways: the account is `disabled` (logged `app/models.py:888`) **or** it has a non-null `delete_on` (scheduled deletion, logged `app/models.py:891`). Crucially, this gate sits *after* the E502 `contact.user.is_active()` gate (`email_handler.py:990`), and `is_active()` returns true whenever `delete_on is None`. Section C exploits that asymmetry to reach E504 through the canonical path:

- **C1 — `disabled=True`, `delete_on=None`:** `is_active()=True` (passes the E502 gate) but `can_send_or_receive()=False` → **E504**. Observed: `RESULT delivered=False code='550 SL E504 Account disabled' (== status.E504)`, `EmailLog: None`, `outbound stored SendRequest count = 0`, `SentAlert 4->4` (no alert on this path), with the log line `email_handler.py:1008 handle_reply() - User <User 1 ...> cannot send emails`.
- **C2 — `disabled=False`, `delete_on=now`:** again `is_active()=True`, `can_send_or_receive()=False` (`app/models.py:891` `... is scheduled to be deleted`) → **E504**, identical downstream state (`EmailLog: None`, outbound 0, `SentAlert 4->4`).
- **C3 — restore:** `disabled=False`, `delete_on=None` → `is_active()=True`, `can_send_or_receive()=True`, returning the seeded owner to a deliverable state so the invariant subset (§8.8) is reproducible run-to-run.

This closes the F-07 gap of "no E504 runtime output": E504 is reached here through the real `handle_reply()` entry point (not a bypass), with before/after account state (`disabled`/`delete_on`/`is_active()`/`can_send_or_receive()`) printed on each drive.

### 8.6 Adversarial `is_reverse_alias()` predicate — the routing gate under hostile input

`is_reverse_alias()` (`app/email_utils.py:1155-1163`) is the routing predicate that decides, for an inbound recipient, whether the message enters the reply path at all (it performs its own `Contact.get_by(reply_email=address)`). Section D feeds it seven hostile inputs directly and asserts the `Contact` rowcount is unchanged before/after:

- empty string, `' OR '1'='1' --@sl.local`, `x'; DROP TABLE contact;--@sl.local`, `a@sl.local\r\nBcc: attacker@evil.test` (CRLF), `réply中文@sl.local` (Unicode), and a 5009-character overlong local-part all return a plain boolean `False` — **no SQL injection, no CRLF effect, no crash**. The SQLAlchemy `filter_by(reply_email=...)` binds the value as a query **parameter**, so quote/keyword payloads are treated as literal data.
- the null-byte input `a\x00b@sl.local` raises `ValueError: 'A string literal cannot contain NUL (0x00) characters.'` — the driver rejects the NUL **before** any query executes (safe; no injection/execution).
- `Contact rowcount before=2 after=2 (unchanged)` — the predicate performs no writes under any of these inputs.

Observed conclusion: the routing predicate is injection-safe and side-effect-free; hostile recipients are rejected as "not a reverse alias" (or rejected at the driver) rather than mutating state or altering routing.

### 8.7 Adversarial `replace_header_when_reply()` — the second lookup site under hostile headers

`replace_header_when_reply()` (`email_handler.py:345`) is the second `Contact.get_by(reply_email=...)` site (§8.3). It strips `\r` (`email_handler.py:355`) and `\n` (`email_handler.py:356-357`) from each parsed address, then either rewrites a reverse-alias to its contact `website_email` or raises `NonReverseAliasInReplyPhase` (`app/errors.py:36`; raised at `email_handler.py:372`). Section E exercises it with four header conditions and asserts the `Contact` rowcount is unchanged:

- **E1 (canonical positive):** a `To` header carrying both the alias address and the real reverse-alias `f07-shared@sl.local` is rewritten to `CA <ca@nowhere.net>` — the reverse-alias is replaced by the contact's `website_email` and the bare alias address is skipped (`email_handler.py:380` `Replace To header ...`).
- **E2 (modern policy defense):** the modern `email.message.EmailMessage` policy **refuses** to hold a header value containing CR/LF, raising `ValueError 'Header values may not contain linefeed or carriage return characters'` at set-time. The pipeline's own message object therefore cannot carry an injected header in the first place (defense-in-depth).
- **E3 (compat32 folded CR/LF):** a `compat32`-parsed folded header whose raw value is `['first@evil.test,\r\n\tsecond@evil.test']` (`has_CR=True has_LF=True`) is processed far enough that the `\r`/`\n` strip runs and the function reaches address parsing, where it raises `NonReverseAliasInReplyPhase` (caught). The warning log at `email_handler.py:366` shows the value **after** stripping (`headers:['first@evil.test,\tsecond@evil.test']`) — no CR/LF survives into any rewritten header.
- **E4 (compat32 adversarial):** a `compat32` `To` header `['"x"; DROP TABLE contact;--...@evil.test']` likewise raises `NonReverseAliasInReplyPhase` (caught) — no crash, no write.
- `Contact rowcount before=2 after=2 (unchanged)` across E1–E4.

Observed conclusion: the second lookup site neither executes injected SQL nor propagates injected headers; malformed/hostile addresses are stripped of CR/LF and then rejected as non-reverse, with no `Contact` mutation.

### 8.8 Stability across runs (F-07 secondary matrix)

Per the methodology (run at scale; confirm stability across at least two runs), the full harness was executed **twice** in fresh processes. A deterministic invariant subset — the `RESULT`/`EmailLog`/`SentAlert`/DB-delta/status-code/rowcount lines, excluding the per-run-varying fields (logger timestamps, PID, randomly-suffixed alias local-parts, `handle_reply()` wall-clock ms, and the generated `sl_message_id`) — is **byte-identical** across both runs (47 lines each, both hashing to `7da7c509…`).

**Command:**

```
docker exec sl_app bash -lc 'cd /tmp/qafix/out; diff f07_inv1.txt f07_inv2.txt && echo "(no differences)"; sha256sum f07_inv1.txt f07_inv2.txt'
```

**Output (complete, unedited):**

```
(no differences)
7da7c509046c40db042f9757c2b5a4d1c39c1de332019ad86118237fe386681d  f07_inv1.txt
7da7c509046c40db042f9757c2b5a4d1c39c1de332019ad86118237fe386681d  f07_inv2.txt
```

The empty `diff` and matching SHA-256 confirm that the status codes (E200/E214/E504), the resolved owner, the alert recipients, and the before/after DB state for every scenario in §8.4–§8.7 are stable run-to-run; only cosmetic per-run fields differ.

---

### 8.9 Security testing — adversarial recipient / header matrix (F-04, SEC-A)

This section adds the **canonical malicious-recipient matrix** required by F-04. Every probe is driven through the **real top-level entry point** `email_handler.handle(envelope, msg)` (`email_handler.py:1945`) — *not* a bypassing helper — with the hostile string placed in `envelope.rcpt_tos[0]` (the recipient the inbound SMTP server hands the application). The six probe classes exercised in this security testing matrix are: **SQL metacharacter** injection (`sql metachar`) as an OR tautology and as a `DROP TABLE` statement, **CRLF** header injection (`crlf`), a **Unicode** local-part, an **overlong** local-part (`overlong`, 309 characters), and a **null-byte** (`\x00`) injection. (In the harness's own summary labels the control bytes are shown escaped — e.g. `a\x00b@sl.local`, `a@sl.local\r\nBcc: …` — while the *real* bytes, NUL and CRLF included, are what is placed on `rcpt_tos` and handed to `handle()`; SimpleLogin's own DEBUG log lines below show the raw bytes as the application logged them.)

For each probe the harness records (a) which phase `handle()` routed the mail into, (b) the returned SMTP status, (c) whether any `EmailLog` was created for the message, (d) the outbound-store count, and (e) the `(contact, alias, users)` rowcounts **before and after** the call — so any injection that created, dropped, or modified a row would be visible as a rowcount delta. After the matrix, a fresh connection issues `select count(*) from contact` to prove the `contact` table still exists (i.e. the `DROP TABLE` payload was inert).

**Command:**

```
docker exec sl_app bash -lc '. /app/venv/bin/activate && set -a && . /tmp/sl_env.sh && set +a && cd /tmp && python observe_sec.py'
```

**Output (complete, unedited; run 1 of 2 — this single run prints both the SEC-A matrix and the SEC-B cross-user-disclosure section analysed in §8.10; stability across runs is machine-verified at the end of §8.10):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 03:54:36,501 - SL - DEBUG - 17345 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 03:54:40,871 - SL - INFO - 17345 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 03:54:40,874 - SL - INFO - 17345 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 03:54:40,875 - SL - INFO - 17345 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 03:54:40,876 - SL - DEBUG - 17345 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 03:54:41,157 - SL - INFO - 17345 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 03:54:41,175 - SL - DEBUG - 17345 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_word590@sl.local
2026-07-14 03:54:41,183 - SL - INFO - 17345 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
========== SECTION SEC-A: ADVERSARIAL RECIPIENTS through canonical email_handler.handle() ==========
seed: owner.id=1 owner.email='owner@mailbox.test' alias_o.id=2 legit reverse-alias='sec-legit@sl.local'
baseline rowcounts (contact,alias,users) = (1, 2, 1)

2026-07-14 03:54:41,196 - SL - DEBUG - 17345 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-14 03:54:41,197 - SL - DEBUG - 17345 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:owner@mailbox.test, rcpt_tos:["'or'1'='1'--@sl.local"], header_from:ext@nowhere.net, header_to:sec-legit@sl.local, cc:None, reply-to:None, message_id:<sec-a0@sl.local>, client_ip:None, headers:[('From', 'ext@nowhere.net'), ('To', 'sec-legit@sl.local'), ('Message-ID', '<sec-a0@sl.local>'), ('Subject', 'sec probe'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 03:54:41,201 - SL - DEBUG - 17345 - "/app/email_handler.py:2202" - handle() -  - Forward phase owner@mailbox.test(ext@nowhere.net) -> 'or'1'='1'--@sl.local
2026-07-14 03:54:41,207 - SL - DEBUG - 17345 - "/app/email_handler.py:545" - handle_forward() -  - alias 'or'1'='1'--@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 03:54:41,212 - SL - INFO - 17345 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() -  - Cannot auto-create custom domain alias for 'or'1'='1'--@sl.local because there's no custom domain for sl.local
2026-07-14 03:54:41,213 - SL - INFO - 17345 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() -  - Cannot auto-create 'or'1'='1'--@sl.local since it has no directory separator
2026-07-14 03:54:41,213 - SL - DEBUG - 17345 - "/app/email_handler.py:551" - handle_forward() -  - alias 'or'1'='1'--@sl.local cannot be created on-the-fly, return 550
--- [SEC-A0 sql_or] rcpt_to=' OR '1'='1' --@sl.local ---
    handle() returned SMTP status = '550 SL E515 Email not exist'
    routed EmailLog for this Message-ID = None
    outbound stored SendRequest count = 0
    rowcounts before=(1, 2, 1) after=(1, 2, 1)  UNCHANGED=True
2026-07-14 03:54:41,219 - SL - DEBUG - 17345 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-14 03:54:41,220 - SL - DEBUG - 17345 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:owner@mailbox.test, rcpt_tos:["x';droptablecontact;--@sl.local"], header_from:ext@nowhere.net, header_to:sec-legit@sl.local, cc:None, reply-to:None, message_id:<sec-a1@sl.local>, client_ip:None, headers:[('From', 'ext@nowhere.net'), ('To', 'sec-legit@sl.local'), ('Message-ID', '<sec-a1@sl.local>'), ('Subject', 'sec probe'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 03:54:41,224 - SL - DEBUG - 17345 - "/app/email_handler.py:2202" - handle() -  - Forward phase owner@mailbox.test(ext@nowhere.net) -> x';droptablecontact;--@sl.local
2026-07-14 03:54:41,229 - SL - DEBUG - 17345 - "/app/email_handler.py:545" - handle_forward() -  - alias x';droptablecontact;--@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 03:54:41,234 - SL - DEBUG - 17345 - "/app/email_handler.py:551" - handle_forward() -  - alias x';droptablecontact;--@sl.local cannot be created on-the-fly, return 550
--- [SEC-A1 sql_drop] rcpt_to=x'; DROP TABLE contact;--@sl.local ---
    handle() returned SMTP status = '550 SL E515 Email not exist'
    routed EmailLog for this Message-ID = None
    outbound stored SendRequest count = 0
    rowcounts before=(1, 2, 1) after=(1, 2, 1)  UNCHANGED=True
2026-07-14 03:54:41,238 - SL - DEBUG - 17345 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-14 03:54:41,239 - SL - DEBUG - 17345 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:owner@mailbox.test, rcpt_tos:['a@sl.local\r bcc:attacker@evil.test'], header_from:ext@nowhere.net, header_to:sec-legit@sl.local, cc:None, reply-to:None, message_id:<sec-a2@sl.local>, client_ip:None, headers:[('From', 'ext@nowhere.net'), ('To', 'sec-legit@sl.local'), ('Message-ID', '<sec-a2@sl.local>'), ('Subject', 'sec probe'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 03:54:41,243 - SL - DEBUG - 17345 - "/app/email_handler.py:2202" - handle() -  - Forward phase owner@mailbox.test(ext@nowhere.net) -> a@sl.local
 bcc:attacker@evil.test
2026-07-14 03:54:41,248 - SL - DEBUG - 17345 - "/app/email_handler.py:545" - handle_forward() -  - alias a@sl.local
 bcc:attacker@evil.test not exist. Try to see if it can be created on the fly
2026-07-14 03:54:41,248 - SL - DEBUG - 17345 - "/app/email_handler.py:551" - handle_forward() -  - alias a@sl.local
 bcc:attacker@evil.test cannot be created on-the-fly, return 550
--- [SEC-A2 crlf] rcpt_to=a@sl.local\r\nBcc: attacker@evil.test ---
    handle() returned SMTP status = '550 SL E515 Email not exist'
    routed EmailLog for this Message-ID = None
    outbound stored SendRequest count = 0
    rowcounts before=(1, 2, 1) after=(1, 2, 1)  UNCHANGED=True
2026-07-14 03:54:41,253 - SL - DEBUG - 17345 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-14 03:54:41,254 - SL - DEBUG - 17345 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:owner@mailbox.test, rcpt_tos:['réply中文@sl.local'], header_from:ext@nowhere.net, header_to:sec-legit@sl.local, cc:None, reply-to:None, message_id:<sec-a3@sl.local>, client_ip:None, headers:[('From', 'ext@nowhere.net'), ('To', 'sec-legit@sl.local'), ('Message-ID', '<sec-a3@sl.local>'), ('Subject', 'sec probe'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 03:54:41,257 - SL - DEBUG - 17345 - "/app/email_handler.py:2202" - handle() -  - Forward phase owner@mailbox.test(ext@nowhere.net) -> réply中文@sl.local
2026-07-14 03:54:41,262 - SL - DEBUG - 17345 - "/app/email_handler.py:545" - handle_forward() -  - alias réply中文@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 03:54:41,262 - SL - DEBUG - 17345 - "/app/email_handler.py:551" - handle_forward() -  - alias réply中文@sl.local cannot be created on-the-fly, return 550
--- [SEC-A3 unicode] rcpt_to=réply中文@sl.local ---
    handle() returned SMTP status = '550 SL E515 Email not exist'
    routed EmailLog for this Message-ID = None
    outbound stored SendRequest count = 0
    rowcounts before=(1, 2, 1) after=(1, 2, 1)  UNCHANGED=True
2026-07-14 03:54:41,266 - SL - DEBUG - 17345 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-14 03:54:41,267 - SL - DEBUG - 17345 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:owner@mailbox.test, rcpt_tos:['aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@sl.local'], header_from:ext@nowhere.net, header_to:sec-legit@sl.local, cc:None, reply-to:None, message_id:<sec-a4@sl.local>, client_ip:None, headers:[('From', 'ext@nowhere.net'), ('To', 'sec-legit@sl.local'), ('Message-ID', '<sec-a4@sl.local>'), ('Subject', 'sec probe'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 03:54:41,271 - SL - DEBUG - 17345 - "/app/email_handler.py:2202" - handle() -  - Forward phase owner@mailbox.test(ext@nowhere.net) -> aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@sl.local
2026-07-14 03:54:41,276 - SL - DEBUG - 17345 - "/app/email_handler.py:545" - handle_forward() -  - alias aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 03:54:41,276 - SL - DEBUG - 17345 - "/app/email_handler.py:551" - handle_forward() -  - alias aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@sl.local cannot be created on-the-fly, return 550
--- [SEC-A4 overlong] rcpt_to=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa...(309 chars) ---
    handle() returned SMTP status = '550 SL E515 Email not exist'
    routed EmailLog for this Message-ID = None
    outbound stored SendRequest count = 0
    rowcounts before=(1, 2, 1) after=(1, 2, 1)  UNCHANGED=True
2026-07-14 03:54:41,280 - SL - DEBUG - 17345 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
--- [SEC-A5 null_byte] rcpt_to=a\x00b@sl.local ---
    handle() RAISED ValueError: 'A string literal cannot contain NUL (0x00) characters.' (input rejected before any injection/execution)
    routed EmailLog for this Message-ID = None
    outbound stored SendRequest count = 0
    rowcounts before=(1, 2, 1) after=(1, 2, 1)  UNCHANGED=True

SEC-A final rowcounts (contact,alias,users) = (1, 2, 1)  == baseline (1, 2, 1) ? True
`contact` table survived DROP-TABLE payload (count(*) succeeded above) = True

========== SECTION SEC-B: CROSS-USER E214 DISCLOSURE (shared reply_email mis-resolves to user_B) ==========
2026-07-14 03:54:44,664 - SL - INFO - 17345 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 03:54:44,666 - SL - INFO - 17345 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 03:54:44,667 - SL - INFO - 17345 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 03:54:44,668 - SL - DEBUG - 17345 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 03:54:44,933 - SL - INFO - 17345 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 03:54:45,193 - SL - INFO - 17345 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
seed: user_b.id=1 ('userb@mailbox.test') alias_b.id=1 ; user_a.id=2 ('usera@mailbox.test') alias_a.id=2
      two contacts share reply_email='dup-disclose@sl.local' : contact_b.id=1(user_b) contact_a.id=2(user_a)
      Contact.get_by(reply_email=SHARED).first() -> id=1 alias_id=1 user_id=1  (resolves user_B? True)
      REPLY will be sent by user_A (mail_from='usera@mailbox.test') who legitimately owns the OTHER duplicate (contact_a)

--- [SEC-B1 default control: alias_b.disable_email_spoofing_check=False] ---
2026-07-14 03:54:45,228 - SL - DEBUG - 17345 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-14 03:54:45,229 - SL - DEBUG - 17345 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:usera@mailbox.test, rcpt_tos:['dup-disclose@sl.local'], header_from:cb@nowhere.net, header_to:dup-disclose@sl.local, cc:None, reply-to:None, message_id:<sec-b1@sl.local>, client_ip:None, headers:[('From', 'cb@nowhere.net'), ('To', 'dup-disclose@sl.local'), ('Message-ID', '<sec-b1@sl.local>'), ('Subject', 'reply from user_A'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 03:54:45,233 - SL - DEBUG - 17345 - "/app/email_handler.py:2196" - handle() -  - Reply phase usera@mailbox.test(cb@nowhere.net) -> dup-disclose@sl.local
2026-07-14 03:54:45,234 - SL - INFO - 17345 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 03:54:45,238 - SL - WARNING - 17345 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: usera@mailbox.test. msg from header: cb@nowhere.net, reverse-alias dup-disclose@sl.local, <Alias 1 simplelogin-newsletter.word725@sl.local> <User 1 Test User userb@mailbox.test> <Contact 1 cb@nowhere.net 1>
2026-07-14 03:54:45,255 - SL - DEBUG - 17345 - "/app/app/email_utils.py:303" - send_email() -  - send email to userb@mailbox.test, subject 'Attempt to use your alias simplelogin-newsletter.word725@sl.local from usera@mailbox.test'
2026-07-14 03:54:45,262 - SL - DEBUG - 17345 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Attempt to use your alias simplelogin-newsletter.word725@sl.local from usera@mailbox.test', from '"noreply@sl.local" <noreply@sl.local>' to 'userb@mailbox.test'
    (1) WRONG Contact resolved : contact.id=1 alias_id=1 user_id=1 (user_B) -- NOT the sender user_A's contact
    (2) WRONG owner            : alias.user=user_1 (user_B) but reply mail_from='usera@mailbox.test' (user_A)
    (3) handle() status        : '250 SL E214 Unauthorized for using reverse alias' (==E214? True)
    (4) successful forward?    : NO (E214 before EmailLog.create/forward)
    (5) EmailLog ownership     : None
    (3) E214 alert -> recipient='userb@mailbox.test'  subject='Attempt to use your alias simplelogin-newsletter.word725@sl.local from usera@mailbox.test'
    (6) notification ownership : SentAlert id=1 user_id=1 alert_type='reverse_alias_unknown_mailbox' (owned by WRONG user_B)
    NEW SentAlert rows = 1 ; outbound count = 1
    DISCLOSURE: user_A's mailbox 'usera@mailbox.test' is exposed in the alert delivered to WRONG user_B ('userb@mailbox.test').

--- [SEC-B2 [non-canonical] fallback: alias_b.disable_email_spoofing_check=True] ---
2026-07-14 03:54:45,275 - SL - DEBUG - 17345 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-14 03:54:45,276 - SL - DEBUG - 17345 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:usera@mailbox.test, rcpt_tos:['dup-disclose@sl.local'], header_from:cb@nowhere.net, header_to:dup-disclose@sl.local, cc:None, reply-to:None, message_id:<sec-b2@sl.local>, client_ip:None, headers:[('From', 'cb@nowhere.net'), ('To', 'dup-disclose@sl.local'), ('Message-ID', '<sec-b2@sl.local>'), ('Subject', 'reply from user_A (spoof off)'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 03:54:45,280 - SL - DEBUG - 17345 - "/app/email_handler.py:2196" - handle() -  - Reply phase usera@mailbox.test(cb@nowhere.net) -> dup-disclose@sl.local
2026-07-14 03:54:45,284 - SL - INFO - 17345 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-14 03:54:45,285 - SL - WARNING - 17345 - "/app/email_handler.py:1023" - handle_reply() -  - ignore unknown sender to reverse-alias usera@mailbox.test: <Alias 1 simplelogin-newsletter.word725@sl.local> -> <Contact 1 cb@nowhere.net 1>
2026-07-14 03:54:45,288 - SL - DEBUG - 17345 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 cb@nowhere.net 1>, <User 1 Test User userb@mailbox.test>, <Mailbox 1 userb@mailbox.test>
2026-07-14 03:54:45,293 - SL - DEBUG - 17345 - "/app/email_handler.py:1171" - handle_reply() -  - From header is simplelogin-newsletter.word725@sl.local
2026-07-14 03:54:45,295 - SL - DEBUG - 17345 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: dup-disclose@sl.local, new: CB <cb@nowhere.net>
2026-07-14 03:54:45,295 - SL - DEBUG - 17345 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-14 03:54:45,297 - SL - DEBUG - 17345 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178400128529.17345.14284414563895045917.1@sl.local>
2026-07-14 03:54:45,303 - SL - DEBUG - 17345 - "/app/email_handler.py:1212" - handle_reply() -  - send email from simplelogin-newsletter.word725@sl.local to cb@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-14 03:54:45,306 - SL - DEBUG - 17345 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply from user_A (spoof off)', from 'simplelogin-newsletter.word725@sl.local' to 'CB <cb@nowhere.net>'
    (3) handle() status        : '250 Message accepted for delivery' (==E200? True)
    (4) successful forward?    : YES
    (5) EmailLog ownership     : id=1 contact_id=1 alias_id=1 user_id=1 (WRONG user_B; sender was user_A)
    (4) forwarded outbound -> envelope_to='cb@nowhere.net' (contact_b.website_email; user_A's reply reaches user_B's contact)
    NEW SentAlert rows = 0 ; outbound count = 1
    [non-canonical] label: this path requires disable_email_spoofing_check=True (a per-alias opt-out), mirrors section 10 CASE 2.

DONE
```

**SEC-A result matrix** (all values quoted verbatim from the output above):

| # | Probe class | `rcpt_to` input | Routed phase | `handle()` result | Rows before→after | Injection effect |
|---|-------------|-----------------|--------------|-------------------|-------------------|------------------|
| A0 | SQL metacharacter (OR tautology) | `' OR '1'='1' --@sl.local` | Forward | `550 SL E515 Email not exist` | `(1, 2, 1)`→`(1, 2, 1)` | None |
| A1 | SQL metacharacter (`DROP TABLE`) | `x'; DROP TABLE contact;--@sl.local` | Forward | `550 SL E515 Email not exist` | `(1, 2, 1)`→`(1, 2, 1)` | None — `contact` table survived (`count(*)` succeeded) |
| A2 | CRLF header injection | `a@sl.local\r\nBcc: attacker@evil.test` | Forward | `550 SL E515 Email not exist` | `(1, 2, 1)`→`(1, 2, 1)` | None — no `Bcc` smuggled; whole string treated as one non-existent alias |
| A3 | Unicode local-part | `réply中文@sl.local` | Forward | `550 SL E515 Email not exist` | `(1, 2, 1)`→`(1, 2, 1)` | None |
| A4 | Overlong local-part (309 chars) | `aaaa…@sl.local` | Forward | `550 SL E515 Email not exist` | `(1, 2, 1)`→`(1, 2, 1)` | None |
| A5 | Null byte | `a\x00b@sl.local` | rejected pre-routing | `ValueError: A string literal cannot contain NUL (0x00) characters.` | `(1, 2, 1)`→`(1, 2, 1)` | None — psycopg2 rejects the NUL before any query executes |

**Interpretation (grounded in the output).** No adversarial recipient reaches the reply-resolution path at all: every syntactically-malformed address fails the `is_reverse_alias()` gate (§8.6) and is routed to `handle_forward()` (`email_handler.py:2202`), where the non-existent alias yields `550 SL E515 Email not exist` (`email_handler.py:551`). Because `Contact.get_by(reply_email=…)` and every other lookup use SQLAlchemy **parametrized** queries, the SQL metacharacter payloads are passed as *data*, never concatenated into SQL — the `contact` table is intact after the `DROP TABLE` attempt, and the final rowcounts equal the baseline `(1, 2, 1)` (`… == baseline (1, 2, 1) ? True`). The CRLF payload is carried as a single opaque local-part: SimpleLogin logs the raw `\r\n` bytes verbatim, but no `Bcc` header is injected into any outbound message (outbound count stays 0). The null-byte address is rejected by the PostgreSQL driver with a clean `ValueError` **before** any SQL executes, so it too changes no rows. This is the adversarial-input half of F-04: the lookup is injection-safe and the reply path is unreachable by malformed recipients.

---

### 8.10 Cross-user privacy and disclosure — the six-way separation (F-04, SEC-B)

F-04 additionally requires an **explicit separation** of the six distinct outcomes that a shared-`reply_email` mis-resolution produces, and specifically calls out the **cross-user privacy** leak the prior draft missed: *the E214 owner-alert is delivered to the wrong user and its subject discloses the replying user's mailbox address.* The SEC-B section of the §8.9 output above drives exactly this condition through the canonical `handle()` entry point. Two contacts on aliases owned by **different** users share `reply_email='dup-disclose@sl.local'`; `Contact.get_by(reply_email=SHARED).first()` resolves the row owned by **user_B** (`id=1 … user_id=1`), while the reply is actually sent by **user_A** (`mail_from='usera@mailbox.test'`), who legitimately owns the *other* duplicate contact.

The output separates the six outcomes explicitly under two conditions: **SEC-B1** is the canonical default (`disable_email_spoofing_check=False`); **SEC-B2** is the `[non-canonical]` per-alias opt-out (`disable_email_spoofing_check=True`) that mirrors §10 CASE 2 and is reached only when an alias owner has explicitly disabled the spoof-check.

| # | Separated outcome | SEC-B1 — canonical default (spoof-check **ON**) | SEC-B2 — `[non-canonical]` opt-out (spoof-check **OFF**) |
|---|-------------------|-------------------------------------------------|----------------------------------------------------------|
| (1) | **Wrong `Contact`** resolved | `contact.id=1 alias_id=1 user_id=1` (user_B) — *not* sender user_A's `contact.id=2` | same wrong `contact.id=1` (user_B) |
| (2) | **Wrong owner** derived (`alias.user`) | `user_1` (user_B); reply `mail_from` is user_A | `user_1` (user_B); reply `mail_from` is user_A |
| (3) | **E214 alert** / `handle()` status | `250 SL E214 Unauthorized for using reverse alias` | `250 Message accepted for delivery` (E200) |
| (4) | **Successful forward** | NO — E214 returned before `EmailLog.create`/forward | YES — outbound `envelope_to='cb@nowhere.net'` (user_B's contact) |
| (5) | **`EmailLog` ownership** | None (no `EmailLog` created) | `EmailLog id=1 contact_id=1 alias_id=1 user_id=1` — **user_B**, though sender was user_A |
| (6) | **Notification ownership** (`SentAlert`) | `SentAlert id=1 user_id=1` — **user_B** | none (0 new `SentAlert`) |

**The cross-user disclosure (the privacy defect F-04 flags).** In the canonical SEC-B1 case the reply from user_A hits the wrong alias (user_B's), fails the mailbox-authorization check, and `handle_unknown_mailbox()` (`email_handler.py:1390`) sends an owner-alert to `user.email` of the **resolved** alias — i.e. to **`userb@mailbox.test`** — with subject `Attempt to use your alias simplelogin-newsletter.<rand>@sl.local from usera@mailbox.test`. The harness states this verbatim: `DISCLOSURE: user_A's mailbox 'usera@mailbox.test' is exposed in the alert delivered to WRONG user_B ('userb@mailbox.test').` So a security-relevant notification about user_A's activity — **including user_A's real mailbox address** — is delivered to an unrelated user_B, and the `SentAlert` audit row is even *owned* by user_B (outcome 6). This is a genuine cross-user privacy leak, distinct from (and additional to) the wrong-user *forwarding* documented in §10: here no forward occurs, yet user_A's identity still leaks to user_B via the alert. In the `[non-canonical]` SEC-B2 opt-out, the mail is instead **forwarded** and the resulting `EmailLog` is attributed to the wrong user_B (outcome 5) — the audit-ownership counterpart of the same root cause.

**Stability across runs.** The full `observe_sec.py` harness was executed **twice** in fresh processes. A deterministic invariant subset — the status codes, rowcounts, resolved-owner ids, `EmailLog`/`SentAlert` ownership, and the disclosure recipient/sender lines, excluding per-run-varying fields (logger timestamps, PID, and the randomly-suffixed alias local-part that appears inside the alert subject) — is **byte-identical** across both runs (51 lines each, both hashing to `03bad162…`).

**Command:**

```
docker exec sl_app bash -lc 'cd /tmp/qafix/out; diff sec_inv1.txt sec_inv2.txt && echo "(no differences)"; sha256sum sec_inv1.txt sec_inv2.txt'
```

**Output (complete, unedited):**

```
(no differences)
03bad1626761527187cc91375d26d6daa2763281cee46e06d7d4daea777584c0  sec_inv1.txt
03bad1626761527187cc91375d26d6daa2763281cee46e06d7d4daea777584c0  sec_inv2.txt
```

The empty `diff` and matching SHA-256 confirm that the resolution (wrong contact / wrong owner), the E214 status, the disclosure recipient (`userb@mailbox.test`) and disclosed sender (`usera@mailbox.test`), and the `EmailLog`/`SentAlert` ownership are stable run-to-run; only cosmetic per-run fields differ.

---
## 9. SimpleLogin reverse-alias concept — intended invariant vs. schema reality

The official SimpleLogin documentation defines the reverse-alias and its intended uniqueness:

- "*A reverse-alias is unique for each sender and alias*" ([SimpleLogin Docs — Reverse alias](https://simplelogin.io/docs/getting-started/reverse-alias/)). This is the **intended invariant**: one `reply_email` corresponds to exactly one `(alias, contact)` pair.
- "*When you send an email to a reverse-alias from your personal email, the email will be sent from your alias to the contact*" ([SimpleLogin FAQ](https://simplelogin.io/faq/)), and — per the documented send-email procedure (choose the alias to send from, then enter the contact's address, which creates a reverse-alias for that contact) — a distinct reverse-alias is created for each alias you send from and each contact you send to, i.e., one per `(alias, contact)` pair (paraphrased) ([SimpleLogin Docs — Send emails from your alias](https://simplelogin.io/docs/getting-started/send-email/)).

The reply path relies on this intended one-to-one mapping when it derives the alias/user/mailbox from the single contact returned by `Contact.get_by(reply_email=...)`. **But the database does not enforce that invariant** (§6.1): with no `UNIQUE` constraint on `reply_email`, two contacts on aliases owned by different users can share one `reply_email`, and the unordered `.first()` then resolves an application-unordered row (§4.1, §6.3). The gap between the documented intent (unique per sender+alias) and the unenforced schema (no `UNIQUE`) is precisely the crux of the wrong-user question.

The framework/database semantics underpinning the ordering analysis are grounded in official sources:

- **SQLAlchemy 1.3** — `Query.first()` emits `LIMIT 1` and returns the first yielded row ([Query API](https://docs.sqlalchemy.org/en/13/orm/query.html)); without `ORDER BY` on more than one match "*it is not deterministic which rows will actually be returned*" and an `ORDER BY` on a unique column is required ([SQLAlchemy 1.3 FAQ — ORDER BY with LIMIT](https://docs.sqlalchemy.org/en/13/faq/ormconfiguration.html#why-is-order-by-required-with-limit-especially-with-subqueryload)).
- **PostgreSQL 15** — "*If sorting is not chosen, the rows will be returned in an unspecified order. The actual order in that case will depend on the scan and join plan types and the order on disk, but it must not be relied on*" ([PostgreSQL 15 §7.5 Sorting Rows](https://www.postgresql.org/docs/15/queries-order.html)); a `SELECT` without `ORDER BY`/`LIMIT` has no guaranteed row order ([PostgreSQL 15 SELECT](https://www.postgresql.org/docs/15/sql-select.html)).

---

## 10. CRITICAL wrong-user scenario — default control vs. `[non-canonical]` fallback (complete, unedited output)

This is the detailed evidence behind the Lead Answer (§1). `observe_wronguser.py` seeds the wrong-user data condition in insertion order **B-then-A** (so `.first()` resolves `user_B`'s contact), then drives the **canonical** `handle_reply()` twice against the **same** seeded rows with **identical** `mail_from = user_A`'s mailbox: once under the **default** spoof control, once under the **`[non-canonical]`** spoof-disabled fallback. Five facets are distinguished per case: (a) application acceptance, (b) reply forwarded?, (c) stored outbound / external target, (d) alert generation, (e) `EmailLog` persistence, (f) confirmed SMTP delivery.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_wronguser.py 2>&1'
```

**Output (complete, unedited; the machine-verified run-2 stability comparison follows immediately below):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:27:17,686 - SL - DEBUG - 5813 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 18:27:21,897 - SL - INFO - 5813 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-13 18:27:21,899 - SL - INFO - 5813 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-13 18:27:21,901 - SL - INFO - 5813 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-13 18:27:21,902 - SL - DEBUG - 5813 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:27:22,183 - SL - INFO - 5813 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 18:27:22,444 - SL - INFO - 5813 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 18:27:22,458 - SL - DEBUG - 5813 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_test874@sl.local
2026-07-13 18:27:22,465 - SL - INFO - 5813 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 18:27:22,475 - SL - DEBUG - 5813 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_word861@sl.local
2026-07-13 18:27:22,482 - SL - INFO - 5813 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== SEED (insertion order B-then-A; .first() should resolve user_B) ===
shared_reply='dup-reply-wrong@sl.local'  mail_from will be user_A='usera@mailbox.test'
user_a.id=1 alias_a.id=3 | user_b.id=2 alias_b.id=4
1st-inserted contact_b.id=1 (user_B) ; 2nd-inserted contact_a.id=2 (user_A)
[non-canonical] Contact.get_by(reply_email=...) -> id=1 user_id=2 (user_B)
=== CASE 1 [canonical, DEFAULT control: alias.disable_email_spoofing_check=False] ===
alias_b.disable_email_spoofing_check = False
2026-07-13 18:27:22,501 - SL - INFO - 5813 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 18:27:22,504 - SL - WARNING - 5813 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: usera@mailbox.test. msg from header: sender@nowhere.net, reverse-alias dup-reply-wrong@sl.local, <Alias 4 list_word861@sl.local> <User 2 Test User userb@mailbox.test> <Contact 1 b@nowhere.net 4>
2026-07-13 18:27:22,522 - SL - DEBUG - 5813 - "/app/app/email_utils.py:303" - send_email() -  - send email to userb@mailbox.test, subject 'Attempt to use your alias list_word861@sl.local from usera@mailbox.test'
2026-07-13 18:27:22,529 - SL - DEBUG - 5813 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Attempt to use your alias list_word861@sl.local from usera@mailbox.test', from '"noreply@sl.local" <noreply@sl.local>' to 'userb@mailbox.test'
(a) application acceptance : delivered=False code='250 SL E214 Unauthorized for using reverse alias'  (==E214? True, ==E200? False)
(b) reply forwarded?       : NO EmailLog and NO reply SendRequest to the external contact (path returned E214 before EmailLog.create/forward)
(c) stored outbound (this run) count=1 -> the unknown-mailbox ALERT, NOT a reply forward:
      ALERT SendRequest envelope_to='userb@mailbox.test' subject='Attempt to use your alias list_word861@sl.local from usera@mailbox.test'
(d) alert generation       : new SentAlert rows=1
      SentAlert id=1 user_id=2(user_B) to_email='userb@mailbox.test' alert_type='reverse_alias_unknown_mailbox'
(e) EmailLog persisted?    : NO (path stopped at E214 before EmailLog.create)
(f) confirmed SMTP delivery: NONE (NOT_SEND_EMAIL=True). The only outbound is the alert to user_B's mailbox; the reply itself is rejected, so user_A's reply is NOT forwarded to any external contact.
=== CASE 2 [NON-CANONICAL fallback: alias_b.disable_email_spoofing_check=True] ===
alias_b.disable_email_spoofing_check = True  (non-default per-alias flag)
2026-07-13 18:27:22,545 - SL - INFO - 5813 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 18:27:22,546 - SL - WARNING - 5813 - "/app/email_handler.py:1023" - handle_reply() -  - ignore unknown sender to reverse-alias usera@mailbox.test: <Alias 4 list_word861@sl.local> -> <Contact 1 b@nowhere.net 4>
2026-07-13 18:27:22,549 - SL - DEBUG - 5813 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 b@nowhere.net 4>, <User 2 Test User userb@mailbox.test>, <Mailbox 2 userb@mailbox.test>
2026-07-13 18:27:22,554 - SL - DEBUG - 5813 - "/app/email_handler.py:1171" - handle_reply() -  - From header is list_word861@sl.local
2026-07-13 18:27:22,556 - SL - DEBUG - 5813 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: dup-reply-wrong@sl.local, new: B <b@nowhere.net>
2026-07-13 18:27:22,556 - SL - DEBUG - 5813 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 18:27:22,558 - SL - DEBUG - 5813 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178396724255.5813.15871454277701825390.1@sl.local>
2026-07-13 18:27:22,562 - SL - WARNING - 5813 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-13 18:27:22,565 - SL - DEBUG - 5813 - "/app/email_handler.py:1212" - handle_reply() -  - send email from list_word861@sl.local to b@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 18:27:22,568 - SL - DEBUG - 5813 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply subject', from 'list_word861@sl.local' to 'B <b@nowhere.net>'
(a) application acceptance : delivered=True code='250 Message accepted for delivery'  (==E200? True)
(b) stored send request    : count=1
      SendRequest envelope_from='sl.lmysyibrfqqdemzygi4dmn25.ddr6o3tn7t3do@sl.local' envelope_to='b@nowhere.net' msg[From]='list_word861@sl.local' msg[To]='B <b@nowhere.net>'
(c) external Contact target: 'b@nowhere.net'  (contact_B.website_email='b@nowhere.net')
(d) alert generation       : new SentAlert rows=0
(e) EmailLog persisted?    : YES id=1 contact_id=1 alias_id=4 user_id=2(user_B(WRONG - not the sender/mail_from owner)) mailbox_id=2 is_reply=True
(f) confirmed SMTP delivery: NONE — NOT_SEND_EMAIL=True: mail_sender.send() logs and returns True WITHOUT calling _send_to_smtp (app/mail_sender.py:130-136). '250 accepted' = application acceptance only.
DONE
```

- **Seed.** `shared_reply='dup-reply-wrong@sl.local'`; insertion B-then-A makes `contact_b.id=1` (user_B) the first row; `[non-canonical] Contact.get_by(...) -> id=1 user_id=2 (user_B)` confirms which row `.first()` resolves.
- **CASE 1 — default control (`disable_email_spoofing_check=False`) → E214, no forward.** `get_mailbox_from_mail_from()` finds no mailbox of user_A authorized for user_B's alias, so `handle_reply()` invokes `handle_unknown_mailbox()` (`email_handler.py:1032`; the function's own internal `LOG.w` warning is emitted at `:1393`, shown in the output above) and returns `(a) delivered=False code='250 SL E214 Unauthorized for using reverse alias'`. `(b)` **no `EmailLog` and no reply send request** — the path returns E214 before `EmailLog.create`/forward. `(c)` the single stored outbound is the **unknown-mailbox alert** to `userb@mailbox.test` (subject `Attempt to use your alias ... from usera@mailbox.test`), **not** a reply forward. `(d)` a new `SentAlert` row is created (`id=1 user_id=2(user_B) alert_type='reverse_alias_unknown_mailbox'`). `(e)` **no `EmailLog`**. `(f)` no confirmed SMTP (`NOT_SEND_EMAIL=True`); the only outbound is the alert to user_B, and user_A's reply is **not** forwarded anywhere. This is an **access-control boundary**.
- **CASE 2 — `[non-canonical]` fallback (`disable_email_spoofing_check=True`) → wrong-user selection, logging, and enqueue.** The spoof check is skipped: `ignore unknown sender ... <Alias 4 ...> -> <Contact 1 b@nowhere.net 4>` (`email_handler.py:1023`), then `Create <EmailLog 1> for <Contact 1 b@nowhere.net 4>, <User 2 ... userb@mailbox.test>, <Mailbox 2 ...>` (`email_handler.py:1051`), header rewrite (`:380`), and send (`:1212`). `(a) delivered=True code='250 Message accepted for delivery'` (E200). `(b)` one stored `SendRequest` with `envelope_to='b@nowhere.net'`. `(c)` external target `b@nowhere.net` = `contact_B.website_email`. `(d)` **no** new `SentAlert`. `(e)` **`EmailLog id=1 contact_id=1 alias_id=4 user_id=2 (user_B — WRONG, not the sending mailbox's owner) mailbox_id=2 is_reply=True`.** `(f)` no confirmed SMTP (`NOT_SEND_EMAIL=True`): the `'250 accepted'` is application acceptance and enqueue only.

**Causal conclusion.** The wrong-user *resolution* is caused by the unordered `.first()` over a non-unique `reply_email` (§4.1, §6.1–§6.3). Whether that mis-resolution becomes a wrong-user **forward** depends on the alias's spoof control: **default** settings convert it into an E214 rejection plus an alert to the resolved (wrong) user (CASE 1); the **`[non-canonical]`** spoof-disabled fallback lets it become an actual forward logged against the wrong user (CASE 2). In neither case is external SMTP delivery confirmed under this harness configuration. The CASE 1 alerting consequence is generalized in **§8.4**, whose five-way sender-authorization matrix shows that *every* unauthorized `mail_from` (a different legitimate user, an unrelated verified user, an unknown external sender, or a malformed address) yields E214 with the alert disclosed to the **resolved** alias owner — so when the shared `reply_email` mis-resolves, the disclosure lands on the wrong user regardless of who actually replied.

**Run-2 stability (machine-verified, not asserted).** The identical harness was executed in a **second, fresh OS process**; the two captures (`observe_wronguser.run1.out`, `observe_wronguser.run2.out`) were compared after masking only the fields that are *legitimately* per-run-varying — logger timestamps + PID, the randomly-generated alias local-parts, and the random reverse-alias token in `envelope_from`:

```
$ maskdoc() { sed -E \
    "s/^20[0-9]{2}-[0-9]{2}-[0-9]{2} [0-9:,]+ - SL - [A-Z]+ - [0-9]+ - /<TS> - SL - LEVEL - <PID> - /; \
     s/[a-z]+_[a-z]+[0-9]+@sl\.local/<RANDOM_ALIAS>@sl.local/g; \
     s/sl\.[a-z0-9]+\.[a-z0-9]+@sl\.local/<RANDOM_REVERSE_ALIAS>@sl.local/g; \
     s/<[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+@sl\.local>/<SL_MESSAGE_ID>/g; \
     s/20[0-9]{2}-[0-9]{2}-[0-9]{2}T[0-9:]+\.[0-9]+\+00:00/<DELETE_ON>/g" "$1"; }
$ diff <(maskdoc observe_wronguser.run1.out) <(maskdoc observe_wronguser.run2.out); echo "exit=$?"
exit=0
$ sha256sum <(maskdoc observe_wronguser.run1.out) <(maskdoc observe_wronguser.run2.out)
b62b738a0e3f12e513aa1c93f0e54bc281a09d300c50d24bfd11daf0fc68761b  /dev/fd/63
b62b738a0e3f12e513aa1c93f0e54bc281a09d300c50d24bfd11daf0fc68761b  /dev/fd/62
```

`diff` produced **no output (exit 0)** and both masked runs share one SHA256, so every semantic value (status codes, resolved identities, and persisted state shown above) is **byte-identical across two fresh processes**; only the enumerated timing/random fields differ.

---
## 11. Per-claim evidence appendix

### 11.1 Factual claims → `file:line` (canonical source at commit `2cd6ee777f8c`, re-confirmed at runtime)

| Claim | Grounding |
|-------|-----------|
| Reply address = inbound recipient | `reply_email = rcpt_to` — `email_handler.py:972` |
| Domain gate / E501 | `email_handler.py:977-981` (`endswith(EMAIL_DOMAIN)`, `SLDomain.get_by`, `status.E501`) |
| Inbound normalization | `reply_email = normalize_reply_email(reply_email)` — `email_handler.py:984`; body `app/email_validation.py:25-38`; `_ALLOWED_CHARS` `app/email_validation.py:9` |
| Primary contact lookup | `contact = Contact.get_by(reply_email=reply_email)` — `email_handler.py:986` |
| E502 no contact / inactive user | `email_handler.py:988-989` / `990-992`; `User.is_active` `app/models.py:766-769` |
| `.first()` with no `ORDER BY` | `ModelMixin.get_by` — `app/models.py:82-84` |
| Alias/user/mailbox derived from contact | `alias = contact.alias` `:994`; `user = alias.user` `:1004`; `get_mailbox_from_mail_from` `:1019` |
| Spoof branch (default E214 vs fallback) | `email_handler.py:1019-1034` (`disable_email_spoofing_check` `:1021`; `handle_unknown_mailbox` invoke `:1032`; `status.E214` `:1034`); `handle_unknown_mailbox` def `:1390`, internal warning `LOG.w` `:1393` |
| `EmailLog` fields incl. `user_id=contact.user_id` | `EmailLog.create(...)` — `email_handler.py:1042-1050` |
| Second lookup site | `replace_header_when_reply` def `email_handler.py:345`; second `Contact.get_by` `:364`; invoked `:1179`(TO)/`:1181`(CC); log `:380/:383` |
| Routing hub dispatch | `handle` def `email_handler.py:1945`; `==>> Handle` `:1980`; `Reply phase ...` `:2196`; `is_reverse_alias` reuse `:2166`/`:2195` |
| No `UNIQUE` on `reply_email` | column `app/models.py:1899` (`index=True`); only `uq_contact(alias_id, website_email)` `app/models.py:1875`; index `unique=False` `migrations/versions/2021_071310_78403c7b8089_.py:22`; runtime `pg_constraint`/`pg_indexes` §6.1(b) |
| Best-effort guard (exactly 3 checks) | `available_sl_email` — `app/models.py:1425-1432` |
| Guard call sites (all three) | `app/email_utils.py:1150`, `app/models.py:1458`, `app/models.py:1706` |
| `is_reverse_alias` two clauses | `app/email_utils.py:1156-1163` |
| Reverse-alias minting | `generate_reply_email` — `app/email_utils.py:1103`; guard call `:1150` |
| Default mailbox = user email | `User.create`: `mb = Mailbox.create(user_id=user.id, email=user.email, verified=True)` `app/models.py:611`; `user.default_mailbox_id = mb.id` `:613` |
| App bootstrap | `create_app` — `server.py:139` |
| dotenv precedence / `NOT_SEND_EMAIL` | `load_dotenv` `app/config.py:69/71` (override defaults False); `NOT_SEND_EMAIL` `app/config.py:91` |
| Store-instead-of-send / no SMTP under `NOT_SEND_EMAIL` | `app/mail_sender.py:130-136` (send logs+returns without `_send_to_smtp`) |

### 11.2 Behavioral claims → command + output (cross-reference)

| Behavioral claim | Command shown in | Output block |
|------------------|------------------|--------------|
| Runtime = Python 3.10.18 / PostgreSQL 15.13 / Redis PONG | §2.1 | §2.1 |
| Effective `EMAIL_DOMAIN`/`DB_URI`/`NOT_SEND_EMAIL` + dotenv grounding | §2.2 | §2.2 |
| Container detached at `2cd6ee...` + 5 dirty files; destination branch/HEAD | §2.3 | §2.3 |
| Duplicate `reply_email` persists (count=2); canonical happy-path resolution + EmailLog + SendRequest; diagnostics | §4.2 | §4.2 |
| `handle()` routes reverse-alias → `handle_reply` (E200) | §4.3 | §4.3 |
| No `UNIQUE`: repo-wide search + runtime catalog (exit 0) | §6.1 | §6.1(a)/(b) |
| Exact `available_sl_email` body + `is_reverse_alias` two clauses + guard bypass via duplicate seed; 3 call sites | §6.2 | §6.2 |
| Same-input distribution (4 seeds + 8 drives) incl. E214 default vs wrong-user fallback; ctid + EXPLAIN | §7.1–§7.4 | §7.1–§7.4 |
| CRITICAL wrong-user: CASE 1 default E214 (+alert), CASE 2 fallback wrong-user log | §10 | §10 |
| E501 / E502(no contact) / E502(inactive) / `is_reverse_alias` | §8.1 | §8.1 |
| Normalization: inbound many-to-one + exact-equality lookup | §8.2 | §8.2 |
| Second lookup site rewrites TO/CC to two distinct contacts | §8.3 | §8.3 |

### 11.3 Full source of every temporary harness (reproduced so results survive deletion)

All thirteen scripts are reproduced verbatim (they live outside the repository and were deleted afterward — absence proof in §13). Each of the twelve Python harnesses shares the fail-fast bootstrap shown in §2.7; the thirteenth, `contact_schema.sql`, is a plain `psql` script (no bootstrap). The exact invocation precedes each source.

**`_bootstrap.py`** — shared allowlist/`current_database`/`pg_trgm` bootstrap (imported inline by the harnesses; shown once):

```python
# Shared corrected bootstrap fragment (documented inline in each harness).
import os
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
_DSN = os.environ.get("DB_URI", "")
if _DSN not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={_DSN!r} is not an allowlisted disposable test DB\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()

with engine.connect() as _c:
    _dbname = _c.execute("select current_database()").scalar()
if _dbname != "test":
    sys.stderr.write(f"REFUSING TO RUN: current_database()={_dbname!r} != 'test'\n")
    sys.exit(3)

with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")


def truncate_clean_slate():
    """Clean slate (deterministic IDs, only this run's rows). Safe: allowlist +
    current_database() guards already ran. Uses engine.begin() so the TRUNCATE
    commits and holds no lingering lock."""
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )
```

**`observe_reply.py`** — canonical baseline + normalized-value **provenance** via non-mutating `sys.settrace` (§3–§5). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_reply.py 2>&1'
```

```python
"""observe_reply.py — CANONICAL reply-path baseline + normalized-value PROVENANCE.

Drives the real inbound reply entry point email_handler.handle_reply(envelope, msg,
rcpt_to) and reports the resolved/normalized reply_email captured from handle_reply()'s
OWN frame local via a non-mutating sys.settrace hook (NOT from a direct helper call).

sys.settrace only OBSERVES frames; it mutates no product code. Two canonical calls:

  CASE 1 (baseline, raw==normalized): rcpt_to='dup-reply-core@sl.local'. Two Contacts
     owned by different users share this reply_email (insertion order A-then-B).
     mail_from=user_A's mailbox => correct-owner happy path (E200).

  CASE 2 (normalization PROVENANCE, raw!=normalized): rcpt_to='dup#norm#core@sl.local'
     ('#' is NOT in app/email_validation.py:9 _ALLOWED_CHARS, so :25 normalize maps it
     to '_'). The seeded Contact carries the NORMALIZED reply_email 'dup_norm_core@sl.local'.
     The trace therefore VISIBLY changes at the Contact.get_by line (email_handler.py:986),
     proving the reported value is the handler local AFTER the :984 normalize step — not
     the raw rcpt_to and not a direct helper call.
"""
import os
import sys
import time

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
_DSN = os.environ.get("DB_URI", "")
if _DSN not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={_DSN!r} is not an allowlisted disposable test DB\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()

with engine.connect() as _c:
    _dbname = _c.execute("select current_database()").scalar()
if _dbname != "test":
    sys.stderr.write(f"REFUSING TO RUN: current_database()={_dbname!r} != 'test'\n")
    sys.exit(3)

with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog, SentAlert
from app.email import status, headers
from app.email_utils import is_reverse_alias
from app.email_validation import normalize_reply_email
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN, NOT_SEND_EMAIL
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


# --- NON-MUTATING tracer over handle_reply()'s own frame ---
_HR_CODE = email_handler.handle_reply.__code__
_WITNESS_LINES = {986}   # Contact.get_by line: reply_email holds the post-:984 normalized value here
_repl_trace = []


def _local_tracer(frame, event, arg):
    if frame.f_code is _HR_CODE and event == "line":
        v = frame.f_locals.get("reply_email", "<unset>")
        changed = (not _repl_trace) or (_repl_trace[-1][1] != v)
        witness = frame.f_lineno in _WITNESS_LINES
        if changed or witness:
            rec = (frame.f_lineno, v)
            if not _repl_trace or _repl_trace[-1] != rec:
                _repl_trace.append(rec)
    return _local_tracer


def _global_tracer(frame, event, arg):
    if frame.f_code is _HR_CODE:
        return _local_tracer
    return None


def drive_reply(label, rcpt_to, mail_from, to_alias_email, mid, user_a, user_b):
    """Run ONE canonical handle_reply() call under the non-mutating tracer; print evidence."""
    global _repl_trace
    _repl_trace = []
    msg = EmailMessage()
    msg["From"] = "sender@nowhere.net"
    msg["To"] = to_alias_email
    msg["Message-ID"] = mid
    msg["Subject"] = "reply subject"
    msg.set_content("hello body")
    envelope = Envelope()
    envelope.mail_from = mail_from
    envelope.rcpt_tos = [rcpt_to]

    mail_sender.purge_stored_emails()
    alert_before = Session.query(SentAlert).count()

    print(f"=== [{label}] CANONICAL CALL: email_handler.handle_reply(envelope, msg, rcpt_to) ===")
    print(f"rcpt_to (raw)                 = {rcpt_to!r}")
    print(f"envelope.mail_from            = {mail_from!r}")
    print(f"Message-ID                    = {mid!r}")
    print(f"[non-canonical] normalize_reply_email({rcpt_to!r}) = {normalize_reply_email(rcpt_to)!r}"
          f"  (direct helper cross-check; NOT the canonical evidence)")
    _t0 = time.monotonic()
    sys.settrace(_global_tracer)
    delivered, code = email_handler.handle_reply(envelope, msg, rcpt_to)
    sys.settrace(None)
    _elapsed = time.monotonic() - _t0
    print(f"RESULT delivered={delivered!r} code={code!r}")
    print(f"code==status.E200? {code == status.E200}  ==E214? {code == status.E214}  ==E502? {code == status.E502}")

    print(f"--- [{label}] CANONICAL handler-local reply_email (non-mutating sys.settrace) ---")
    for ln, v in _repl_trace:
        tag = "  <- post-:984 normalized value used by Contact.get_by(:986)" if ln in _WITNESS_LINES else ""
        print(f"  email_handler.py:{ln}  reply_email = {v!r}{tag}")
    _canon = _repl_trace[-1][1] if _repl_trace else "<none>"
    print(f"CANONICAL normalized reply_email (handler local at :986) = {_canon!r}")

    el = EmailLog.get_by(message_id=mid)
    if el:
        print(f"PERSISTED EmailLog (message_id={mid!r}): id={el.id} contact_id={el.contact_id} "
              f"alias_id={el.alias_id} user_id={el.user_id} mailbox_id={el.mailbox_id} is_reply={el.is_reply}")
        print(f"  correlate: raw rcpt_to={rcpt_to!r} -> handler-local normalized={_canon!r} "
              f"-> resolved contact_id={el.contact_id} -> forwarding user_id={el.user_id} "
              f"(user_a.id={user_a.id}, user_b.id={user_b.id})")
    else:
        print(f"No EmailLog for message_id={mid!r} (non-success path).")

    stored = mail_sender.get_stored_emails()
    print(f"stored SendRequest count = {len(stored)}")
    for sr in stored:
        print(f"  SendRequest envelope_from={sr.envelope_from!r} envelope_to={sr.envelope_to!r} "
              f"msg[From]={sr.msg[headers.FROM]!r}")
    alert_after = Session.query(SentAlert).count()
    print(f"NEW SentAlert rows during call: {alert_after - alert_before}")
    print(f"handle_reply() wall-clock: {_elapsed*1000:.1f} ms")
    print()


mail_sender.store_emails_instead_of_sending(True)

with app.app_context():
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()

    shared_reply = f"dup-reply-core@{EMAIL_DOMAIN}"          # CASE 1: raw == normalized
    norm_reply = f"dup_norm_core@{EMAIL_DOMAIN}"             # CASE 2: the NORMALIZED stored value
    raw_reply = f"dup#norm#core@{EMAIL_DOMAIN}"              # CASE 2: the RAW rcpt_to ('#' -> '_')

    user_a = create_new_user(email="usera@mailbox.test")
    user_b = create_new_user(email="userb@mailbox.test")
    Session.commit()
    alias_a = Alias.create_new_random(user_a)
    alias_b = Alias.create_new_random(user_b)
    alias_a2 = Alias.create_new_random(user_a)
    Session.commit()

    # CASE 1 duplicate pair (A-then-B) sharing shared_reply
    contact_a = Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id,
                               website_email="a@nowhere.net", name="A", reply_email=shared_reply)
    contact_b = Contact.create(user_id=alias_b.user_id, alias_id=alias_b.id,
                               website_email="b@nowhere.net", name="B", reply_email=shared_reply)
    # CASE 2 single contact carrying the NORMALIZED reply_email
    contact_n = Contact.create(user_id=alias_a2.user_id, alias_id=alias_a2.id,
                               website_email="n@nowhere.net", name="N", reply_email=norm_reply)
    Session.commit()

    print("=== SEED (insertion order A-then-B) ===")
    print(f"EMAIL_DOMAIN={EMAIL_DOMAIN!r}  NOT_SEND_EMAIL={NOT_SEND_EMAIL!r}")
    print(f"CASE1 shared_reply={shared_reply!r}")
    print(f"CASE2 raw_reply={raw_reply!r}  norm_reply(stored)={norm_reply!r}")
    print(f"user_a.id={user_a.id} email={user_a.email!r} alias_a.id={alias_a.id} alias_a.email={alias_a.email!r}")
    print(f"user_b.id={user_b.id} email={user_b.email!r} alias_b.id={alias_b.id} alias_b.email={alias_b.email!r}")
    print(f"alias_a2.id={alias_a2.id} alias_a2.email={alias_a2.email!r} (also owned by user_a)")
    print(f"contact_a.id={contact_a.id} (alias_a,user_a)  contact_b.id={contact_b.id} (alias_b,user_b)  "
          f"contact_n.id={contact_n.id} (alias_a2,user_a)")

    print("=== PROVE CASE1 DUPLICATE PERSISTED (no UNIQUE constraint blocked it) ===")
    rows = Contact.filter_by(reply_email=shared_reply).all()
    print(f"rows sharing reply_email={shared_reply!r}: count={len(rows)}")
    for r in rows:
        print(f"  Contact id={r.id} alias_id={r.alias_id} user_id={r.user_id} website_email={r.website_email!r}")

    print("=== raw reply-address derivation inputs (email_handler.py:972,977) ===")
    print(f"CASE1 rcpt_to.endswith(EMAIL_DOMAIN) = {shared_reply.endswith(EMAIL_DOMAIN)}")
    print(f"CASE2 rcpt_to.endswith(EMAIL_DOMAIN) = {raw_reply.endswith(EMAIL_DOMAIN)}")
    print()

    drive_reply("CASE1 baseline raw==normalized", shared_reply, user_a.email,
                alias_a.email, "<obs-core-0@sl.local>", user_a, user_b)
    drive_reply("CASE2 provenance raw!=normalized", raw_reply, user_a.email,
                alias_a2.email, "<obs-core-norm-0@sl.local>", user_a, user_b)

    print("NOTE: NOT_SEND_EMAIL=%r => mail_sender.send() logs and returns True WITHOUT calling "
          "_send_to_smtp; no external SMTP delivery is confirmed (app/mail_sender.py:130-136)." % NOT_SEND_EMAIL)
    print("DONE")
```

**`observe_handle.py`** — canonical routing through `email_handler.handle()` (§4.3). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_handle.py 2>&1'
```

```python
"""observe_handle.py — CANONICAL routing hub: email_handler.handle() dispatches a
reverse-alias recipient to handle_reply().

Seeds ONE clean contact (owned by user_A) and drives the real routing hub
email_handler.handle(envelope, msg) — NOT handle_reply directly. handle() at
email_handler.py:2195-2199 tests is_reverse_alias(rcpt_to) and, when True, logs
"Reply phase ..." and calls handle_reply(). This proves the routing claim by
actually invoking handle(), rather than asserting it from is_reverse_alias alone.
"""
import os
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI", "") not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={os.environ.get('DB_URI','')!r} not in disposable allowlist\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()
with engine.connect() as _c:
    if _c.execute("select current_database()").scalar() != "test":
        sys.stderr.write("REFUSING TO RUN: current_database() != 'test'\n")
        sys.exit(3)
with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog
from app.email import status, headers
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


mail_sender.store_emails_instead_of_sending(True)

with app.app_context():
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()

    user_a = create_new_user(email="usera@mailbox.test")
    Session.commit()
    alias_a = Alias.create_new_random(user_a)
    Session.commit()
    reply_email = f"route-check@{EMAIL_DOMAIN}"
    contact_a = Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id,
                               website_email="a@nowhere.net", name="A", reply_email=reply_email)
    Session.commit()

    mid = "<route-0@sl.local>"
    msg = EmailMessage()
    msg["From"] = "a@nowhere.net"
    msg["To"] = reply_email
    msg["Message-ID"] = mid
    msg["Subject"] = "reply via handle()"
    msg["Date"] = "Mon, 13 Jul 2026 00:00:00 +0000"
    msg.set_content("hello via handle")
    env = Envelope()
    env.mail_from = user_a.email
    env.rcpt_tos = [reply_email]

    print("=== CANONICAL ROUTING: email_handler.handle(envelope, msg) (NOT handle_reply directly) ===")
    print(f"reply_email(reverse-alias)={reply_email!r}  mail_from={env.mail_from!r}  Message-ID={mid!r}")
    smtp_status = email_handler.handle(env, msg)
    print(f"handle() returned SMTP status = {smtp_status!r}  (==E200? {smtp_status == status.E200})")
    el = EmailLog.get_by(message_id=mid)
    if el:
        print(f"handle() routed to handle_reply -> EmailLog id={el.id} contact_id={el.contact_id} "
              f"alias_id={el.alias_id} user_id={el.user_id} is_reply={el.is_reply}")
    else:
        print("No EmailLog (routing did not reach a successful reply).")
    print("DONE")
```

**`observe_wronguser.py`** — CRITICAL default-control vs `[non-canonical]` fallback (§10). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_wronguser.py 2>&1'
```

```python
"""observe_wronguser.py — the WRONG-USER condition, DEFAULT control vs [non-canonical] fallback.

Seeds two Contacts sharing one reply_email in insertion order B-then-A, so the
unordered .first() resolves user_B's contact, while the reply is sent from
user_A's mailbox (mail_from = user_A). Then drives the CANONICAL entry point
email_handler.handle_reply() twice against the SAME seeded rows:

  CASE 1 (DEFAULT control, spoof check ON): get_mailbox_from_mail_from() returns
          None -> handle_unknown_mailbox() -> status.E214. The path STOPS before
          EmailLog/send. This is the observed default-control result.
  CASE 2 ([non-canonical] fallback, alias_b.disable_email_spoofing_check=True):
          the None mailbox falls back to alias_b.mailbox (user_B's) -> EmailLog
          logged under user_B -> a SendRequest is stored to contact_B's external
          address. Labeled [non-canonical] because it disables a default control.

The output separates FIVE distinct facets: (a) application acceptance,
(b) stored send request, (c) external Contact target, (d) alert generation,
(e) confirmed SMTP delivery (NONE: NOT_SEND_EMAIL=true short-circuits SMTP).
"""
import os
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
_DSN = os.environ.get("DB_URI", "")
if _DSN not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={_DSN!r} is not an allowlisted disposable test DB\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()

with engine.connect() as _c:
    _dbname = _c.execute("select current_database()").scalar()
if _dbname != "test":
    sys.stderr.write(f"REFUSING TO RUN: current_database()={_dbname!r} != 'test'\n")
    sys.exit(3)

with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog, SentAlert
from app.email import status, headers
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN, NOT_SEND_EMAIL
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


def drive(mail_from, rcpt_to, mid):
    msg = EmailMessage()
    msg["From"] = "sender@nowhere.net"
    msg["To"] = rcpt_to
    msg["Message-ID"] = mid
    msg["Subject"] = "reply subject"
    msg.set_content("hello body")
    env = Envelope()
    env.mail_from = mail_from
    env.rcpt_tos = [rcpt_to]
    mail_sender.purge_stored_emails()
    before = Session.query(SentAlert).count()
    delivered, code = email_handler.handle_reply(env, msg, rcpt_to)
    after = Session.query(SentAlert).count()
    el = EmailLog.get_by(message_id=mid)
    stored = mail_sender.get_stored_emails()
    return delivered, code, el, stored, before, after


mail_sender.store_emails_instead_of_sending(True)

with app.app_context():
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()

    shared_reply = f"dup-reply-wrong@{EMAIL_DOMAIN}"
    user_a = create_new_user(email="usera@mailbox.test")
    user_b = create_new_user(email="userb@mailbox.test")
    Session.commit()
    alias_a = Alias.create_new_random(user_a)
    alias_b = Alias.create_new_random(user_b)
    Session.commit()

    # INSERTION ORDER B-then-A: user_B's contact inserted FIRST -> .first() resolves user_B.
    contact_b = Contact.create(user_id=alias_b.user_id, alias_id=alias_b.id,
                               website_email="b@nowhere.net", name="B", reply_email=shared_reply)
    contact_a = Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id,
                               website_email="a@nowhere.net", name="A", reply_email=shared_reply)
    Session.commit()

    print("=== SEED (insertion order B-then-A; .first() should resolve user_B) ===")
    print(f"shared_reply={shared_reply!r}  mail_from will be user_A={user_a.email!r}")
    print(f"user_a.id={user_a.id} alias_a.id={alias_a.id} | user_b.id={user_b.id} alias_b.id={alias_b.id}")
    print(f"1st-inserted contact_b.id={contact_b.id} (user_B) ; 2nd-inserted contact_a.id={contact_a.id} (user_A)")
    d = Contact.get_by(reply_email=shared_reply)
    print(f"[non-canonical] Contact.get_by(reply_email=...) -> id={d.id} user_id={d.user_id} "
          f"({'user_B' if d.user_id == user_b.id else 'user_A'})")

    # -------- CASE 1: DEFAULT control (spoof check ON) --------
    print("=== CASE 1 [canonical, DEFAULT control: alias.disable_email_spoofing_check=False] ===")
    print(f"alias_b.disable_email_spoofing_check = {alias_b.disable_email_spoofing_check}")
    delivered, code, el, stored, sa_before, sa_after = drive(user_a.email, shared_reply, "<wrong-default-0@sl.local>")
    print(f"(a) application acceptance : delivered={delivered!r} code={code!r}  "
          f"(==E214? {code == status.E214}, ==E200? {code == status.E200})")
    print(f"(b) reply forwarded?       : NO EmailLog and NO reply SendRequest to the external contact "
          f"(path returned E214 before EmailLog.create/forward)")
    print(f"(c) stored outbound (this run) count={len(stored)} -> the unknown-mailbox ALERT, NOT a reply forward:")
    for sr in stored:
        print(f"      ALERT SendRequest envelope_to={sr.envelope_to!r} subject={sr.msg[headers.SUBJECT]!r}")
    print(f"(d) alert generation       : new SentAlert rows={sa_after - sa_before}")
    for a in Session.query(SentAlert).all():
        who = "user_B" if a.user_id == user_b.id else ("user_A" if a.user_id == user_a.id else "?")
        print(f"      SentAlert id={a.id} user_id={a.user_id}({who}) to_email={a.to_email!r} alert_type={a.alert_type!r}")
    print(f"(e) EmailLog persisted?    : {'YES id=%d user_id=%d' % (el.id, el.user_id) if el else 'NO (path stopped at E214 before EmailLog.create)'}")
    print(f"(f) confirmed SMTP delivery: NONE (NOT_SEND_EMAIL={NOT_SEND_EMAIL}). The only outbound is the alert to "
          f"user_B's mailbox; the reply itself is rejected, so user_A's reply is NOT forwarded to any external contact.")

    # -------- CASE 2: [non-canonical] spoof-disabled fallback --------
    print("=== CASE 2 [NON-CANONICAL fallback: alias_b.disable_email_spoofing_check=True] ===")
    alias_b.disable_email_spoofing_check = True
    Session.commit()
    print(f"alias_b.disable_email_spoofing_check = {alias_b.disable_email_spoofing_check}  (non-default per-alias flag)")
    delivered, code, el, stored, sa_before, sa_after = drive(user_a.email, shared_reply, "<wrong-fallback-0@sl.local>")
    print(f"(a) application acceptance : delivered={delivered!r} code={code!r}  (==E200? {code == status.E200})")
    print(f"(b) stored send request    : count={len(stored)}")
    if stored:
        sr = stored[0]
        print(f"      SendRequest envelope_from={sr.envelope_from!r} envelope_to={sr.envelope_to!r} "
              f"msg[From]={sr.msg[headers.FROM]!r} msg[To]={sr.msg[headers.TO]!r}")
    _tgt = repr(stored[0].envelope_to) if stored else "(none)"
    print(f"(c) external Contact target: {_tgt}  (contact_B.website_email={contact_b.website_email!r})")
    print(f"(d) alert generation       : new SentAlert rows={sa_after - sa_before}")
    if el:
        who = "user_B(WRONG - not the sender/mail_from owner)" if el.user_id == user_b.id else "user_A"
        print(f"(e) EmailLog persisted?    : YES id={el.id} contact_id={el.contact_id} alias_id={el.alias_id} "
              f"user_id={el.user_id}({who}) mailbox_id={el.mailbox_id} is_reply={el.is_reply}")
    else:
        print("(e) EmailLog persisted?    : NO")
    print(f"(f) confirmed SMTP delivery: NONE — NOT_SEND_EMAIL={NOT_SEND_EMAIL}: mail_sender.send() logs and returns "
          f"True WITHOUT calling _send_to_smtp (app/mail_sender.py:130-136). '250 accepted' = application acceptance only.")
    print("DONE")
```

**`observe_dist.py`** — same-unchanged-input distribution, seed-once/drive-many (§7). Invocations: `seed <ab|ba> <on|off>` and `drive <N> <mid_prefix>`, each run via the canonical runner shown in §7.0. This is the exact 313-line source that produced every §7 seed and drive block.

```python
"""observe_dist.py — Cross-event distribution under the SAME UNCHANGED INPUT.

Two modes, so the dataset is SEEDED ONCE and then the IDENTICAL input is DRIVEN
many times against those same persisted rows. ONLY the Message-ID varies between
events; From, To, Subject, Date and body are all FIXED constants (see FIXED_* below).

  seed <ab|ba> <on|off>
      Truncate (allowlist-guarded), seed a FIXED dataset: user_A/user_B, one alias
      each, and TWO Contact rows sharing the SAME reply_email (fixed website_emails).
      <ab|ba> controls contact INSERTION ORDER. <on|off> sets both aliases'
      disable_email_spoofing_check. Prints ctid physical order + EXPLAIN (default and
      forced Seq Scan) for the exact lookup query. Persists and STOPS.

  drive <N> <mid_prefix>
      Does NOT reseed. Asserts exactly two contacts share the reply_email (else exit 3).
      Drives handle_reply() N times with an IDENTICAL envelope+message; ONLY the
      Message-ID varies (= "<mid_prefix>-i@sl.local"). Correlates each event by exact
      Message-ID via EmailLog and reports a COMPLETE keyed distribution across every
      dimension (status code, resolved contact/alias/user/mailbox/is_reply OR the
      non-forward reason), plus the notification dimension (new SentAlert rows created
      during this drive, grouped by alert_type/to_email/user_id), the outbound dimension
      (stored SendRequest count grouped by envelope_to), and per-event handle_reply()
      durations (total/mean/min/max ms via a monotonic clock). Distribution counters are
      authoritative and sum to N; the first three events and the last are printed as a
      readable sample only.

Arg validation: invalid args exit non-zero (2 = bad args, 3 = data precondition).
"""
import os
import re
import sys
import time

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI", "") not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={os.environ.get('DB_URI','')!r} not in disposable allowlist\n")
    sys.exit(3)

# ---- bounded/validated CLI parsing BEFORE any heavy import ----
SHARED_REPLY = "dist-shared@sl.local"
FIXED_MAIL_FROM = "usera@mailbox.test"   # user_A's mailbox (fixed across every event)
FIXED_FROM = "someone@nowhere.net"       # message From header (fixed)
FIXED_SUBJECT = "dist reply"             # message Subject (FIXED — does NOT vary per event)
FIXED_DATE = "Mon, 13 Jul 2026 00:00:00 +0000"
FIXED_BODY = "body"
WEB_A = "a-dist@nowhere.net"
WEB_B = "b-dist@nowhere.net"

def usage_exit():
    sys.stderr.write(
        "usage: observe_dist.py seed <ab|ba> <on|off> | drive <N:1..1000> <mid_prefix>\n"
    )
    sys.exit(2)

if len(sys.argv) < 2:
    usage_exit()
MODE = sys.argv[1]
if MODE == "seed":
    if len(sys.argv) != 4:
        usage_exit()
    ORDER = sys.argv[2]
    SPOOF = sys.argv[3]
    if ORDER not in ("ab", "ba"):
        sys.stderr.write(f"invalid order {ORDER!r}: must be 'ab' or 'ba'\n")
        sys.exit(2)
    if SPOOF not in ("on", "off"):
        sys.stderr.write(f"invalid spoof {SPOOF!r}: must be 'on' or 'off'\n")
        sys.exit(2)
elif MODE == "drive":
    if len(sys.argv) != 4:
        usage_exit()
    try:
        N = int(sys.argv[2])
    except ValueError:
        sys.stderr.write(f"invalid N {sys.argv[2]!r}: must be an integer\n")
        sys.exit(2)
    if not (1 <= N <= 1000):
        sys.stderr.write(f"invalid N {N}: must be within 1..1000\n")
        sys.exit(2)
    MID_PREFIX = sys.argv[3]
    if not re.fullmatch(r"[A-Za-z0-9_.-]+", MID_PREFIX):
        sys.stderr.write(f"invalid mid_prefix {MID_PREFIX!r}: must match [A-Za-z0-9_.-]+\n")
        sys.exit(2)
else:
    usage_exit()

from server import create_app
from app.db import Session, engine

app = create_app()
with engine.connect() as _c:
    if _c.execute("select current_database()").scalar() != "test":
        sys.stderr.write("REFUSING TO RUN: current_database() != 'test'\n")
        sys.exit(3)
with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog, SentAlert
from app.email import status
from app.mail_sender import mail_sender
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


mail_sender.store_emails_instead_of_sending(True)


def do_seed():
    with app.app_context():
        truncate_clean_slate()
        add_sl_domains()
        add_proton_partner()
        user_a = create_new_user(email=FIXED_MAIL_FROM)
        Session.commit()
        user_b = create_new_user(email="userb@mailbox.test")
        Session.commit()
        alias_a = Alias.create_new_random(user_a)
        alias_b = Alias.create_new_random(user_b)
        Session.commit()
        flag = (SPOOF == "on")
        alias_a.disable_email_spoofing_check = flag
        alias_b.disable_email_spoofing_check = flag
        Session.commit()

        def mk_a():
            return Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id,
                                  website_email=WEB_A, name="A", reply_email=SHARED_REPLY)

        def mk_b():
            return Contact.create(user_id=alias_b.user_id, alias_id=alias_b.id,
                                  website_email=WEB_B, name="B", reply_email=SHARED_REPLY)

        if ORDER == "ab":
            c1 = mk_a(); Session.commit()
            c2 = mk_b(); Session.commit()
        else:
            c1 = mk_b(); Session.commit()
            c2 = mk_a(); Session.commit()

        print(f"=== SEED mode order={ORDER} spoof={SPOOF} (disable_email_spoofing_check={flag}) ===")
        print(f"shared reply_email={SHARED_REPLY!r}  FIXED_MAIL_FROM={FIXED_MAIL_FROM!r}")
        print(f"user_a.id={user_a.id} alias_a.id={alias_a.id}  user_b.id={user_b.id} alias_b.id={alias_b.id}")
        print(f"insertion #1 -> Contact id={c1.id} alias_id={c1.alias_id} user_id={c1.user_id} website={c1.website_email!r}")
        print(f"insertion #2 -> Contact id={c2.id} alias_id={c2.alias_id} user_id={c2.user_id} website={c2.website_email!r}")

        print("--- physical order (ctid) of rows sharing reply_email ---")
        rows = engine.execute(
            "SELECT ctid, id, alias_id, user_id, website_email FROM contact "
            "WHERE reply_email=%(r)s ORDER BY ctid", {"r": SHARED_REPLY}
        ).fetchall()
        for row in rows:
            print(f"  ctid={row[0]} id={row[1]} alias_id={row[2]} user_id={row[3]} website={row[4]!r}")

        print("--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---")
        for row in engine.execute(
            "EXPLAIN SELECT contact.id FROM contact WHERE contact.reply_email=%(r)s LIMIT 1",
            {"r": SHARED_REPLY}).fetchall():
            print("  " + row[0])

        print("--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---")
        with engine.connect() as conn:
            conn.execute("SET enable_indexscan=off")
            conn.execute("SET enable_bitmapscan=off")
            conn.execute("SET enable_indexonlyscan=off")
            for row in conn.execute(
                "EXPLAIN SELECT contact.id FROM contact WHERE contact.reply_email=%(r)s LIMIT 1",
                {"r": SHARED_REPLY}).fetchall():
                print("  " + row[0])
        print("SEED DONE (persisted; run 'drive' next)")


def do_drive():
    with app.app_context():
        n_rows = Session.query(Contact).filter(Contact.reply_email == SHARED_REPLY).count()
        if n_rows != 2:
            sys.stderr.write(f"DATA PRECONDITION FAILED: expected 2 contacts sharing reply_email, found {n_rows}. "
                             f"Run 'seed' first.\n")
            sys.exit(3)
        purge = getattr(mail_sender, "purge_stored_emails", None)
        if purge:
            purge()

        # Snapshot pre-existing SentAlert ids so we count only rows created during THIS drive.
        alert_ids_before = {a.id for a in Session.query(SentAlert).all()}

        direct = Contact.get_by(reply_email=SHARED_REPLY)

        # The structured report is BUFFERED and flushed as ONE contiguous block AFTER the
        # drive loop, so that NO application framework-log line is ever interleaved with it.
        # A single [live] preamble is printed now so a live viewer sees progress; it is
        # OUTSIDE the extracted "=== DRIVE mode ... DRIVE DONE" report block.
        print(f"[live] driving handle_reply() {N}x (structured report buffered; flushed after the run) ...")
        report = []

        def emit(s):
            report.append(s)

        emit(f"=== DRIVE mode N={N} mid_prefix={MID_PREFIX!r} (dataset NOT reseeded) ===")
        emit(f"[non-canonical] direct Contact.get_by(reply_email={SHARED_REPLY!r}) -> "
             f"id={direct.id} alias_id={direct.alias_id} user_id={direct.user_id} website={direct.website_email!r}")
        emit(f"driving handle_reply() {N}x with IDENTICAL mail_from={FIXED_MAIL_FROM!r}, rcpt_to={SHARED_REPLY!r}, "
             f"From={FIXED_FROM!r}, Subject={FIXED_SUBJECT!r}, Date/body FIXED; ONLY Message-ID varies")

        by_code = {}
        by_full = {}
        durations = []
        for i in range(N):
            mid = f"<{MID_PREFIX}-{i}@sl.local>"
            msg = EmailMessage()
            msg["From"] = FIXED_FROM
            msg["To"] = SHARED_REPLY
            msg["Message-ID"] = mid
            msg["Subject"] = FIXED_SUBJECT
            msg["Date"] = FIXED_DATE
            msg.set_content(FIXED_BODY)
            env = Envelope()
            env.mail_from = FIXED_MAIL_FROM
            env.rcpt_tos = [SHARED_REPLY]
            _t0 = time.monotonic()
            delivered, code = email_handler.handle_reply(env, msg, SHARED_REPLY)
            _dt_ms = (time.monotonic() - _t0) * 1000.0
            Session.commit()
            durations.append(_dt_ms)
            by_code[code] = by_code.get(code, 0) + 1
            el = EmailLog.get_by(message_id=mid)
            if el:
                key = (f"code={code!r}|EmailLog:contact_id={el.contact_id},alias_id={el.alias_id},"
                       f"user_id={el.user_id},mailbox_id={el.mailbox_id},is_reply={el.is_reply}")
                by_full[key] = by_full.get(key, 0) + 1
                if i < 3 or i == N - 1:
                    emit(f"  event mid={mid} delivered={delivered} code={code!r} "
                         f"EmailLog(mid match={el.message_id==mid}) contact_id={el.contact_id} "
                         f"alias_id={el.alias_id} user_id={el.user_id} mailbox_id={el.mailbox_id} is_reply={el.is_reply}")
            else:
                key = f"code={code!r}|EmailLog=None"
                by_full[key] = by_full.get(key, 0) + 1
                if i < 3 or i == N - 1:
                    emit(f"  event mid={mid} delivered={delivered} code={code!r} EmailLog=None "
                         f"(no EmailLog persisted on this path)")

        # ---- STATUS distribution ----
        emit("--- distribution by return code (status) ---")
        for k in sorted(by_code):
            emit(f"  code={k!r}: {by_code[k]}")

        # ---- FULL keyed distribution (contact/alias/user/mailbox/is_reply | EmailLog) ----
        emit("--- FULL keyed distribution (status | EmailLog contact/alias/user/mailbox/is_reply) ---")
        _tot = 0
        for k in sorted(by_full):
            emit(f"  {k}: {by_full[k]}")
            _tot += by_full[k]
        emit(f"  (sum of keyed distribution = {_tot}; expected N = {N}; match = {_tot == N})")

        # ---- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ----
        stored = mail_sender.get_stored_emails()
        by_outbound = {}
        for sr in stored:
            by_outbound[sr.envelope_to] = by_outbound.get(sr.envelope_to, 0) + 1
        emit("--- OUTBOUND distribution (stored SendRequest, grouped by envelope_to) ---")
        emit(f"  total stored SendRequest = {len(stored)}")
        for k in sorted(by_outbound):
            emit(f"  envelope_to={k!r}: {by_outbound[k]}")
        if not by_outbound:
            emit("  (none)")

        # ---- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ----
        new_alerts = [a for a in Session.query(SentAlert).all() if a.id not in alert_ids_before]
        by_alert = {}
        for a in new_alerts:
            k = f"alert_type={a.alert_type!r},to_email={a.to_email!r},user_id={a.user_id}"
            by_alert[k] = by_alert.get(k, 0) + 1
        emit("--- NOTIFICATION distribution (new SentAlert rows created during THIS drive) ---")
        emit(f"  total new SentAlert = {len(new_alerts)}")
        for k in sorted(by_alert):
            emit(f"  {k}: {by_alert[k]}")
        if not by_alert:
            emit("  (none)")

        # ---- DURATIONS (per-event handle_reply(), monotonic clock) ----
        _sum = sum(durations)
        _mean = _sum / len(durations)
        emit("--- per-event handle_reply() durations (monotonic) ---")
        emit(f"  events={len(durations)} total={_sum:.1f} ms mean={_mean:.2f} ms "
             f"min={min(durations):.2f} ms max={max(durations):.2f} ms")
        emit("DRIVE DONE")

        # Flush the buffered, contiguous, framework-log-free report as a single write.
        print("\n".join(report))


if MODE == "seed":
    do_seed()
else:
    do_drive()
```

**`observe_edge.py`** — E501 / E502(no contact) / E502(inactive) / `is_reverse_alias` (§8.1). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_edge.py 2>&1'
```

```python
"""observe_edge.py — Secondary/edge conditions of the reply-resolution path, all via
the canonical handle_reply() entry point (no bypassing). Conditions:
  (1) E501  reply domain not EMAIL_DOMAIN and not an SLDomain   [email_handler.py:977-981]
  (2) E502  no Contact matches the reply_email                  [email_handler.py:986-989]
  (3) E502  contact.user.is_active() is False (soft-deleted)    [email_handler.py:990-992; is_active models.py:766-769]
  (4) is_reverse_alias() True for a seeded reply_email, False otherwise [app/email_utils.py:1156-1158]
(E214 wrong-user alert under DEFAULT control is exercised in observe_wronguser.py CASE 1.)
"""
import os
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI", "") not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={os.environ.get('DB_URI','')!r} not in disposable allowlist\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()
with engine.connect() as _c:
    if _c.execute("select current_database()").scalar() != "test":
        sys.stderr.write("REFUSING TO RUN: current_database() != 'test'\n")
        sys.exit(3)
with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

import arrow
from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact
from app.email import status
from app.email_utils import is_reverse_alias
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


def mk_msg(mid):
    msg = EmailMessage()
    msg["From"] = "someone@nowhere.net"
    msg["To"] = "x@sl.local"
    msg["Message-ID"] = mid
    msg["Subject"] = "edge"
    msg["Date"] = "Mon, 13 Jul 2026 00:00:00 +0000"
    msg.set_content("body")
    return msg


def mk_env(mail_from, rcpt_to):
    env = Envelope()
    env.mail_from = mail_from
    env.rcpt_tos = [rcpt_to]
    return env


mail_sender.store_emails_instead_of_sending(True)

with app.app_context():
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()

    user = create_new_user(email="edge-user@mailbox.test")
    Session.commit()
    alias = Alias.create_new_random(user)
    Session.commit()
    active_reply = f"edge-active@{EMAIL_DOMAIN}"
    Contact.create(user_id=alias.user_id, alias_id=alias.id, website_email="ext@nowhere.net",
                   name="Ext", reply_email=active_reply)
    Session.commit()

    # a soft-deleted user (delete_on in the future -> is_active() False), own alias+contact
    inactive_user = create_new_user(email="edge-inactive@mailbox.test")
    Session.commit()
    inactive_alias = Alias.create_new_random(inactive_user)
    Session.commit()
    inactive_reply = f"edge-inactive-reply@{EMAIL_DOMAIN}"
    Contact.create(user_id=inactive_alias.user_id, alias_id=inactive_alias.id,
                   website_email="ext2@nowhere.net", name="Ext2", reply_email=inactive_reply)
    inactive_user.delete_on = arrow.now().shift(days=1)
    Session.commit()

    print("=== (1) E501: reply domain not EMAIL_DOMAIN and not an SLDomain ===")
    rcpt = "reply@notsl.example"
    d, code = email_handler.handle_reply(mk_env(user.email, rcpt), mk_msg("<edge-e501@sl.local>"), rcpt)
    print(f"rcpt_to={rcpt!r} -> delivered={d} code={code!r}  (==E501? {code == status.E501})")

    print("=== (2) E502: no Contact matches the reply_email ===")
    rcpt = f"no-such-contact@{EMAIL_DOMAIN}"
    d, code = email_handler.handle_reply(mk_env(user.email, rcpt), mk_msg("<edge-e502a@sl.local>"), rcpt)
    print(f"rcpt_to={rcpt!r} -> delivered={d} code={code!r}  (==E502? {code == status.E502})")

    print("=== (3) E502: contact.user.is_active() is False (soft-deleted user) ===")
    iu = Contact.get_by(reply_email=inactive_reply).user
    print(f"inactive user delete_on={iu.delete_on}  is_active()={iu.is_active()}")
    d, code = email_handler.handle_reply(mk_env(inactive_user.email, inactive_reply),
                                         mk_msg("<edge-e502b@sl.local>"), inactive_reply)
    print(f"rcpt_to={inactive_reply!r} -> delivered={d} code={code!r}  (==E502? {code == status.E502})")

    print("=== (4) is_reverse_alias() predicate [app/email_utils.py:1156-1158] ===")
    print(f"is_reverse_alias({active_reply!r})            = {is_reverse_alias(active_reply)}")
    print(f"is_reverse_alias('not-a-reverse-alias@{EMAIL_DOMAIN}') = "
          f"{is_reverse_alias('not-a-reverse-alias@' + EMAIL_DOMAIN)}")
    print("DONE")
```

**`observe_norm.py`** — inbound normalization / exact-equality lookup (§8.2). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_norm.py 2>&1'
```

```python
"""observe_norm.py — Corrected normalization semantics (#4).

normalize_reply_email() [app/email_validation.py:25-38] is applied ONLY to the
INBOUND rcpt_to string at email_handler.py:984; the stored Contact.reply_email is
NOT normalized inside the query (email_handler.py:986 does exact-equality
Contact.get_by(reply_email=<normalized-inbound>)). Therefore:
  * MANY inbound spellings that differ only by disallowed characters collapse onto
    ONE stored key (each disallowed char -> '_').
  * A multi-row match still requires MULTIPLE STORED rows that already share the
    same (normalized) key; inbound normalization alone does not create duplicates.
"""
import os
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI", "") not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={os.environ.get('DB_URI','')!r} not in disposable allowlist\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()
with engine.connect() as _c:
    if _c.execute("select current_database()").scalar() != "test":
        sys.stderr.write("REFUSING TO RUN: current_database() != 'test'\n")
        sys.exit(3)
with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog
from app.email import status
from app.email_validation import normalize_reply_email
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


mail_sender.store_emails_instead_of_sending(True)

STORED = f"norm_key@{EMAIL_DOMAIN}"   # already-normalized: '_' is in _ALLOWED_CHARS
INBOUND_VARIANTS = [
    f"norm_key@{EMAIL_DOMAIN}",   # identity
    f"norm key@{EMAIL_DOMAIN}",   # space  -> '_'
    f"norm#key@{EMAIL_DOMAIN}",   # '#'    -> '_'
    f"norm~key@{EMAIL_DOMAIN}",   # '~'    -> '_'
]

with app.app_context():
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()
    user = create_new_user(email="norm-user@mailbox.test")
    Session.commit()
    alias = Alias.create_new_random(user)
    Session.commit()
    Contact.create(user_id=alias.user_id, alias_id=alias.id, website_email="ext@nowhere.net",
                   name="Ext", reply_email=STORED)
    Session.commit()

    print(f"=== stored Contact.reply_email = {STORED!r} (single row) ===")
    print("--- normalize_reply_email() maps each INBOUND spelling onto the stored key ---")
    for v in INBOUND_VARIANTS:
        nv = normalize_reply_email(v)
        print(f"  inbound={v!r:32} -> normalize_reply_email -> {nv!r}  (==stored? {nv == STORED})")

    print("--- exact-equality proof: query uses normalized INBOUND vs raw STORED ---")
    raw = f"norm key@{EMAIL_DOMAIN}"
    print(f"  Contact.get_by(reply_email={raw!r})            -> {Contact.get_by(reply_email=raw)}")
    print(f"  Contact.get_by(reply_email=normalize({raw!r})) -> "
          f"{Contact.get_by(reply_email=normalize_reply_email(raw))}")

    print("--- canonical handle_reply() with each inbound spelling resolves the ONE stored contact ---")
    for i, v in enumerate(INBOUND_VARIANTS):
        mid = f"<norm-{i}@sl.local>"
        msg = EmailMessage()
        msg["From"] = "someone@nowhere.net"
        msg["To"] = STORED
        msg["Message-ID"] = mid
        msg["Subject"] = f"norm {i}"
        msg["Date"] = "Mon, 13 Jul 2026 00:00:00 +0000"
        msg.set_content("body")
        env = Envelope()
        env.mail_from = user.email
        env.rcpt_tos = [v]
        d, code = email_handler.handle_reply(env, msg, v)
        Session.commit()
        el = EmailLog.get_by(message_id=mid)
        cid = el.contact_id if el else None
        print(f"  inbound={v!r:32} -> delivered={d} code={code!r} EmailLog.contact_id={cid}")

    n = Session.query(Contact).filter(Contact.reply_email == STORED).count()
    print(f"--- rows sharing the stored key = {n} (multi-row match needs >1 STORED row with same key) ---")
    print("DONE")
```

**`observe_avail.py`** — exact `available_sl_email` body, call sites, guard bypass (§6.2). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_avail.py 2>&1'
```

```python
"""observe_avail.py — available_sl_email() guard evidence (#15, #20).

(20) Reproduce the EXACT body of available_sl_email() (inspect.getsource) — it has
     exactly THREE checks: Alias / Contact / DeletedAlias.
(15) Enumerate ALL THREE call sites (grounded, not just the generator):
       app/email_utils.py:1150  generate_reply_email()
       app/models.py:1458       generate_random_alias_email()
       app/models.py:1706       custom-alias creation
     Then PROVE a direct Contact.create() BYPASSES the guard: seed one contact,
     show available_sl_email(reply)==False (would block the generator), yet a direct
     Contact.create() with the SAME reply_email succeeds -> two rows (no UNIQUE
     constraint, guard is only consulted by the generator loop, not at write time).
"""
import os
import sys
import inspect

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI", "") not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={os.environ.get('DB_URI','')!r} not in disposable allowlist\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()
with engine.connect() as _c:
    if _c.execute("select current_database()").scalar() != "test":
        sys.stderr.write("REFUSING TO RUN: current_database() != 'test'\n")
        sys.exit(3)
with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

import app.models as models
from app.models import Alias, Contact, available_sl_email, ModelMixin
from app.email_utils import is_reverse_alias
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


mail_sender.store_emails_instead_of_sending(True)

print("=== EXACT body of available_sl_email() [app/models.py] (inspect.getsource) ===")
src, start = inspect.getsourcelines(available_sl_email)
for off, line in enumerate(src):
    print(f"  {start+off}: {line.rstrip()}")

print("=== EXACT body of ModelMixin.get_by() (the .first() with no ORDER BY) ===")
src, start = inspect.getsourcelines(ModelMixin.get_by.__func__)
for off, line in enumerate(src):
    print(f"  {start+off}: {line.rstrip()}")

print("=== EXACT body of is_reverse_alias() [app/email_utils.py] ===")
src, start = inspect.getsourcelines(is_reverse_alias)
for off, line in enumerate(src):
    print(f"  {start+off}: {line.rstrip()}")

DUP = f"dupe-guard@{EMAIL_DOMAIN}"

with app.app_context():
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()
    user = create_new_user(email="avail-user@mailbox.test")
    Session.commit()
    alias = Alias.create_new_random(user)
    Session.commit()

    print("=== guard bypass: direct Contact.create() ignores available_sl_email() ===")
    print(f"before any contact: available_sl_email({DUP!r}) = {available_sl_email(DUP)}")
    c1 = Contact.create(user_id=alias.user_id, alias_id=alias.id, website_email="one@nowhere.net",
                        name="One", reply_email=DUP)
    Session.commit()
    print(f"after 1st Contact.create: available_sl_email({DUP!r}) = {available_sl_email(DUP)} "
          f"(guard would now block the GENERATOR from choosing this value)")
    # direct create with the SAME reply_email despite guard==False -> no exception, no UNIQUE
    c2 = Contact.create(user_id=alias.user_id, alias_id=alias.id, website_email="two@nowhere.net",
                        name="Two", reply_email=DUP)
    Session.commit()
    n = Session.query(Contact).filter(Contact.reply_email == DUP).count()
    print(f"direct 2nd Contact.create with SAME reply_email SUCCEEDED: c1.id={c1.id} c2.id={c2.id}; "
          f"rows sharing reply_email={n}")
    print("=> available_sl_email() is a generation-time TOCTOU check only; a direct write bypasses it "
          "and no DB UNIQUE constraint prevents the duplicate.")
    print("DONE")
```

**`observe_race.py`** — GENUINE concurrent availability-check / generator race (two OS processes: real check → real commit), backing §6.2.1. The filesystem barrier and the `gen`-mode leader/follower hand-off are **synchronization controls only**; `available_sl_email()`, `generate_reply_email()`, `Contact.create()`, and `Session.commit()` are the real, unmodified application code. SHA256 `34f6d0170d2a9e25be6afdff062e8a01c08f2b3260949b5a67e5238cb65f836f`. Invocation:
```
# seed once (clean slate; two users, each with a default+random alias; NO contacts yet):
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_race.py seed 2>&1'
# two concurrent writers share ONE barrier dir (fixed shared target), then report:
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_race.py attempt p1 /tmp/racebarA1 fixed 0 2>&1' &
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_race.py attempt p2 /tmp/racebarA1 fixed 0 2>&1' &
wait
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_race.py report /tmp/racebarA1 2>&1'
# generator-path collision: fresh barrier dir + mode 'gen' (p1 mints via the real
# generate_reply_email() loop and publishes it; p2 consumes that generator-produced value):
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_race.py attempt p1 /tmp/racebarG gen 0 2>&1' &
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_race.py attempt p2 /tmp/racebarG gen 0 2>&1' &
wait
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_race.py report /tmp/racebarG 2>&1'
```

```python
"""observe_race.py - GENUINE availability-check / generator race (TOCTOU).

Modes:
  seed
      Truncate to a clean slate; create user_a (usera@mailbox.test)+alias_a and
      user_b (userb@mailbox.test)+alias_b. Creates NO contacts (the race creates them).
  attempt <p1|p2> <barrier_dir> <fixed|gen> <seed_int>
      Drive the REAL check-then-create-then-commit path under a synchronized barrier:
        1) target = RACE_SHARED (fixed) OR, in gen mode, p1 mints target via the REAL
           generate_reply_email(...) generator loop and publishes it; p2 consumes that
           generator-produced value (secrets.choice CSPRNG at app/utils.py:L47 makes two
           independent generator calls never collide, so the leader/follower hand-off is
           what presents one generator-produced value to both concurrent writers)
        2) checked = available_sl_email(target)                         # REAL check (read)
        3) BARRIER: write <role>.checked; wait until BOTH p1 and p2 have checked
        4) Contact.create(reply_email=target, ...); Session.commit()    # REAL use (write)
        5) record commit outcome + post-commit rowcount sharing target
      p1 attaches its contact to alias_a; p2 to alias_b (distinct website_email so the
      (alias_id, website_email) unique constraint never fires - isolating reply_email).
      The barrier is only a SYNCHRONIZATION CONTROL to force the check-before-commit
      interleaving; available_sl_email, generate_reply_email, Contact.create, and commit
      are the REAL application code.
  report <barrier_dir>
      Read target from the barrier markers; print all contacts sharing that reply_email
      (count + ctid/id/alias_id/user_id/website), the observed checks and commits, and a verdict.
"""
import os, sys, time, random

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI") not in _ALLOWED_DSNS:
    sys.stderr.write("REFUSING TO RUN: DB_URI %r not in allowlist\n" % os.environ.get("DB_URI"))
    sys.exit(3)

RACE_SHARED = "race-shared@sl.local"
USER_A = "usera@mailbox.test"
USER_B = "userb@mailbox.test"
WEB_A = "racea@nowhere.net"
WEB_B = "raceb@nowhere.net"
GEN_CONTACT_EMAIL = "race-contact@nowhere.net"


def usage_exit():
    sys.stderr.write("usage: observe_race.py seed | attempt <p1|p2> <barrier_dir> <fixed|gen> <seed_int> | report <barrier_dir>\n")
    sys.exit(2)


if len(sys.argv) < 2:
    usage_exit()
MODE = sys.argv[1]

from server import create_app
from app.db import Session, engine

app = create_app()
with engine.connect() as _c:
    if _c.execute("select current_database()").scalar() != "test":
        sys.stderr.write("REFUSING TO RUN: current_database() != 'test'\n")
        sys.exit(3)
with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

from app.models import Alias, Contact, User, available_sl_email
from app.email_utils import generate_reply_email
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


def do_seed():
    with app.app_context():
        truncate_clean_slate()
        add_sl_domains()
        add_proton_partner()
        user_a = create_new_user(email=USER_A); Session.commit()
        user_b = create_new_user(email=USER_B); Session.commit()
        # create_new_user provisions a default alias per user; add a random one too to be safe
        Alias.create_new_random(user_a)
        Alias.create_new_random(user_b)
        Session.commit()
        # print the SAME alias each attempt selects (its user's first alias) so seed and
        # attempt output are consistent
        alias_a = Session.query(Alias).filter(Alias.user_id == user_a.id).first()
        alias_b = Session.query(Alias).filter(Alias.user_id == user_b.id).first()
        print("=== SEED (race) ===")
        print(f"user_a.id={user_a.id} first_alias_a.id={alias_a.id} email={alias_a.email!r}")
        print(f"user_b.id={user_b.id} first_alias_b.id={alias_b.id} email={alias_b.email!r}")
        n = Session.query(Contact).filter(Contact.reply_email == RACE_SHARED).count()
        print(f"contacts sharing {RACE_SHARED!r} at seed = {n}")
        print("SEED DONE")


def _wait_both(barrier_dir, timeout=60.0):
    t0 = time.monotonic()
    while True:
        p1 = os.path.exists(os.path.join(barrier_dir, "p1.checked"))
        p2 = os.path.exists(os.path.join(barrier_dir, "p2.checked"))
        if p1 and p2:
            return True
        if time.monotonic() - t0 > timeout:
            return False
        time.sleep(0.02)


def do_attempt(role, barrier_dir, gmode, seed_int):
    os.makedirs(barrier_dir, exist_ok=True)
    with app.app_context():
        if role == "p1":
            user = User.get_by(email=USER_A); web = WEB_A
        else:
            user = User.get_by(email=USER_B); web = WEB_B
        alias = Session.query(Alias).filter(Alias.user_id == user.id).first()

        print(f"=== RACE attempt role={role} mode={gmode} ===")
        if gmode == "gen":
            # p1 (leader) mints the reverse-alias with the REAL generator loop
            # (available_sl_email is called INSIDE generate_reply_email) and publishes it;
            # p2 (follower) consumes that generator-produced value. random.seed() cannot
            # force a collision because random_string uses secrets.choice (CSPRNG,
            # app/utils.py:L47), so the hand-off is what presents ONE generator-produced
            # value to both concurrent writers.
            gt = os.path.join(barrier_dir, "gen_target.txt")
            if role == "p1":
                target = generate_reply_email(GEN_CONTACT_EMAIL, alias)
                with open(gt, "w") as f:
                    f.write(target + "\n")
                print(f"[{role}] generate_reply_email(...) -> {target!r} (real generator; published to peer)")
            else:
                t0 = time.monotonic()
                while not os.path.exists(gt) and time.monotonic() - t0 < 60.0:
                    time.sleep(0.02)
                target = open(gt).read().strip()
                print(f"[{role}] consumed generator-produced target -> {target!r}")
        else:
            target = RACE_SHARED

        checked = available_sl_email(target)
        print(f"[{role}] target={target!r}")
        print(f"[{role}] available_sl_email(target) BEFORE barrier = {checked}")

        with open(os.path.join(barrier_dir, f"{role}.checked"), "w") as f:
            f.write(target + "\n")
        print(f"[{role}] barrier: wrote {role}.checked; waiting for peer to also check ...")
        both = _wait_both(barrier_dir)
        print(f"[{role}] barrier: both processes checked = {both} - proceeding to Contact.create + commit")

        try:
            c = Contact.create(user_id=user.id, alias_id=alias.id,
                               website_email=web, name=role.upper(), reply_email=target)
            Session.commit()
            outcome = f"OK(id={c.id})"
            print(f"[{role}] Contact.create(reply_email=target, alias_id={alias.id}, website={web!r}) committed id={c.id}")
        except Exception as e:
            Session.rollback()
            outcome = f"EXC({type(e).__name__})"
            print(f"[{role}] Contact.create/commit raised: {type(e).__name__}: {e}")

        cnt = Session.query(Contact).filter(Contact.reply_email == target).count()
        print(f"[{role}] post-commit rows sharing target = {cnt}")
        with open(os.path.join(barrier_dir, f"{role}.committed"), "w") as f:
            f.write(f"checked={checked} outcome={outcome} postcount={cnt}\n")
        print(f"RACE attempt DONE role={role}")


def do_report(barrier_dir):
    with app.app_context():
        target = RACE_SHARED
        for r in ("p1", "p2"):
            p = os.path.join(barrier_dir, f"{r}.checked")
            if os.path.exists(p):
                target = open(p).read().strip()
                break
        rows = engine.execute(
            "SELECT ctid, id, alias_id, user_id, website_email FROM contact "
            "WHERE reply_email=%(t)s ORDER BY ctid", {"t": target}
        ).fetchall()
        print(f"=== RACE report target={target!r} ===")
        print(f"rows sharing reply_email={target!r}: count={len(rows)}")
        for row in rows:
            print(f"  ctid={row[0]} id={row[1]} alias_id={row[2]} user_id={row[3]} website={row[4]!r}")

        def readmark(name):
            p = os.path.join(barrier_dir, name)
            return open(p).read().strip() if os.path.exists(p) else "(absent)"

        p1c = readmark("p1.committed")
        p2c = readmark("p2.committed")
        print(f"p1.checked target : {readmark('p1.checked')}")
        print(f"p2.checked target : {readmark('p2.checked')}")
        print(f"p1.committed      : {p1c}")
        print(f"p2.committed      : {p2c}")
        both_checked_true = ("checked=True" in p1c) and ("checked=True" in p2c)
        two_rows = (len(rows) == 2)
        print(f"VERDICT: both_processes_saw_available=True: {both_checked_true}; "
              f"duplicate_reply_email_committed: {two_rows} "
              f"(no UNIQUE on reply_email blocked the second insert)")
        print("RACE report DONE")


if MODE == "seed":
    do_seed()
elif MODE == "attempt":
    if len(sys.argv) != 6:
        usage_exit()
    role, barrier_dir, gmode = sys.argv[2], sys.argv[3], sys.argv[4]
    try:
        seed_int = int(sys.argv[5])
    except ValueError:
        usage_exit()
    if role not in ("p1", "p2") or gmode not in ("fixed", "gen"):
        usage_exit()
    do_attempt(role, barrier_dir, gmode, seed_int)
elif MODE == "report":
    if len(sys.argv) != 3:
        usage_exit()
    do_report(sys.argv[2])
else:
    usage_exit()
```

**`observe_f07.py`** — five-way E214 sender-authorization matrix, reachable E504, and adversarial `is_reverse_alias()` / `replace_header_when_reply()` (§8.4–§8.8). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_f07.py 2>&1'
```

```python
"""observe_f07.py - F-07 named alternate/error/secondary behavior matrix (canonical).

Drives the REAL inbound reply entry point email_handler.handle_reply(envelope, msg,
rcpt_to) with mail_sender.store_emails_instead_of_sending(True) to capture outbound
without external SMTP. Single run; five sections:

  A) Seed: user_A/B/C each with default mailbox+alias; two contacts cA(aliasA,user_A)
     then cB(aliasB,user_B) sharing ONE reply_email SHARED (user_B is a genuine
     co-owner of the same reverse-alias). .first() resolves cA (lowest ctid) so the
     resolved alias/user is deterministically aliasA/user_A for the matrix.
  B) Five-way E214 sender matrix (rcpt_to=SHARED -> resolves aliasA/user_A): vary
     envelope.mail_from across 5 named classes and capture (delivered,code), the
     matching status constant, EmailLog, outbound (envelope_to+subject+From), and
     NEW SentAlert rows incl. the alert RECIPIENT (cross-user disclosure).
  C) Reachable E504: set user_A.disabled=True (delete_on=None => is_active()=True,
     can_send_or_receive()=False) -> E504; also the delete_on route; then restore.
  D) Adversarial is_reverse_alias() predicate: Unicode/CRLF/overlong/SQLi/null/empty
     inputs -> returns a bool without injection; Contact rowcount unchanged.
  E) Adversarial replace_header_when_reply(): To header with a real reverse-alias
     (replaced), a CRLF-injection payload (\\r/\\n stripped), and a non-reverse
     adversarial address (raises NonReverseAliasInReplyPhase, caught) - no crash;
     Contact rowcount unchanged.
"""
import os
import sys
import time

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
_DSN = os.environ.get("DB_URI", "")
if _DSN not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={_DSN!r} is not an allowlisted disposable test DB\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()

with engine.connect() as _c:
    _dbname = _c.execute("select current_database()").scalar()
if _dbname != "test":
    sys.stderr.write(f"REFUSING TO RUN: current_database()={_dbname!r} != 'test'\n")
    sys.exit(3)

with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

import arrow
from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog, SentAlert, User
from app.email import status, headers
from app.email_utils import is_reverse_alias
from app.email_validation import normalize_reply_email
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN, NOT_SEND_EMAIL
from app.errors import NonReverseAliasInReplyPhase
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


SHARED = f"f07-shared@{EMAIL_DOMAIN}"


def _status_name(code):
    for nm in ("E200", "E201", "E214", "E501", "E502", "E503", "E504"):
        if code == getattr(status, nm):
            return nm
    return "?"


def drive(label, sender_class, mail_from, alias_to_email, mid):
    """One canonical handle_reply() call; print the full named-scenario row."""
    msg = EmailMessage()
    msg["From"] = "external-contact@nowhere.net"
    msg["To"] = alias_to_email
    msg["Message-ID"] = mid
    msg["Subject"] = "reply subject"
    msg.set_content("hello body")
    envelope = Envelope()
    envelope.mail_from = mail_from
    envelope.rcpt_tos = [SHARED]

    el_before = Session.query(EmailLog).count()
    al_before = Session.query(SentAlert).count()
    mail_sender.purge_stored_emails()

    print(f"--- [{label}] sender_class={sender_class} ---")
    print(f"    envelope.mail_from = {mail_from!r}   rcpt_to = {SHARED!r}   Message-ID = {mid!r}")
    _t0 = time.monotonic()
    delivered, code = email_handler.handle_reply(envelope, msg, SHARED)
    _elapsed = (time.monotonic() - _t0) * 1000.0
    print(f"    RESULT delivered={delivered!r} code={code!r}  (== status.{_status_name(code)})")

    el = EmailLog.get_by(message_id=mid)
    if el:
        print(f"    EmailLog: id={el.id} contact_id={el.contact_id} alias_id={el.alias_id} "
              f"user_id={el.user_id} mailbox_id={el.mailbox_id} is_reply={el.is_reply}")
    else:
        print(f"    EmailLog: None (no log persisted on this path)")

    stored = mail_sender.get_stored_emails()
    print(f"    outbound stored SendRequest count = {len(stored)}")
    for sr in stored:
        subj = sr.msg[headers.SUBJECT]
        print(f"      -> envelope_to={sr.envelope_to!r} subject={subj!r} msg[From]={sr.msg[headers.FROM]!r}")
    al_after = Session.query(SentAlert).count()
    print(f"    NEW SentAlert rows during call = {al_after - al_before}")
    print(f"    DB deltas: EmailLog {el_before}->{Session.query(EmailLog).count()}  "
          f"SentAlert {al_before}->{al_after}")
    print(f"    handle_reply() wall-clock = {_elapsed:.1f} ms")
    print()


mail_sender.store_emails_instead_of_sending(True)

with app.app_context():
    # ============================ SECTION A - SEED ============================
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()

    user_a = create_new_user(email="usera@mailbox.test")
    user_b = create_new_user(email="userb@mailbox.test")
    user_c = create_new_user(email="userc@mailbox.test")
    Session.commit()
    alias_a = Session.query(Alias).filter(Alias.user_id == user_a.id).first()
    alias_b = Session.query(Alias).filter(Alias.user_id == user_b.id).first()
    alias_c = Session.query(Alias).filter(Alias.user_id == user_c.id).first()

    # cA first (lowest ctid -> resolved by .first()), then cB sharing the SAME reply_email
    c_a = Contact.create(user_id=user_a.id, alias_id=alias_a.id,
                         website_email="ca@nowhere.net", name="CA", reply_email=SHARED)
    Session.commit()
    c_b = Contact.create(user_id=user_b.id, alias_id=alias_b.id,
                         website_email="cb@nowhere.net", name="CB", reply_email=SHARED)
    Session.commit()

    print("========== SECTION A: SEED ==========")
    print(f"EMAIL_DOMAIN={EMAIL_DOMAIN!r}  NOT_SEND_EMAIL={NOT_SEND_EMAIL!r}  SHARED={SHARED!r}")
    print(f"user_a.id={user_a.id} email={user_a.email!r} alias_a.id={alias_a.id} alias_a.email={alias_a.email!r}")
    print(f"user_b.id={user_b.id} email={user_b.email!r} alias_b.id={alias_b.id} alias_b.email={alias_b.email!r}")
    print(f"user_c.id={user_c.id} email={user_c.email!r} alias_c.id={alias_c.id} alias_c.email={alias_c.email!r}")
    rows = engine.execute(
        "SELECT ctid, id, alias_id, user_id, website_email FROM contact "
        "WHERE reply_email=%(t)s ORDER BY ctid", {"t": SHARED}
    ).fetchall()
    print(f"contacts sharing reply_email={SHARED!r}: count={len(rows)}")
    for r in rows:
        print(f"  ctid={r[0]} id={r[1]} alias_id={r[2]} user_id={r[3]} website={r[4]!r}")
    resolved = Contact.get_by(reply_email=SHARED)
    print(f"Contact.get_by(reply_email=SHARED).first() -> id={resolved.id} alias_id={resolved.alias_id} "
          f"user_id={resolved.user_id} (resolved owner = user_a? {resolved.user_id == user_a.id})")
    print()

    # ==================== SECTION B - FIVE-WAY E214 MATRIX ====================
    print("========== SECTION B: FIVE-WAY E214 SENDER MATRIX (rcpt_to resolves aliasA/user_A) ==========")
    drive("B1 resolved-owner",     "resolved_owner",     user_a.email,            alias_a.email, "<f07-b1@sl.local>")
    drive("B2 other-dup-owner",    "other_dup_owner",    user_b.email,            alias_a.email, "<f07-b2@sl.local>")
    drive("B3 unrelated-verified", "unrelated_verified", user_c.email,            alias_a.email, "<f07-b3@sl.local>")
    drive("B4 unknown-sender",     "unknown_sender",     "stranger@nowhere.test", alias_a.email, "<f07-b4@sl.local>")
    drive("B5 malformed-sender",   "malformed_sender",   "not-an-email",          alias_a.email, "<f07-b5@sl.local>")

    # =================== SECTION C - REACHABLE E504 ===================
    print("========== SECTION C: REACHABLE E504 (resolved alias owner cannot send/receive) ==========")
    print("--- C1: user_A.disabled=True (delete_on=None => is_active()=True, can_send_or_receive()=False) ---")
    user_a.disabled = True
    user_a.delete_on = None
    Session.commit()
    ua = User.get_by(email="usera@mailbox.test")
    print(f"    user_a.disabled={ua.disabled} delete_on={ua.delete_on!r} "
          f"is_active()={ua.is_active()} can_send_or_receive()={ua.can_send_or_receive()}")
    drive("C1 disabled-user", "resolved_owner_disabled", user_a.email, alias_a.email, "<f07-c1@sl.local>")

    print("--- C2: user_A.disabled=False, delete_on=now (scheduled deletion) ---")
    ua = User.get_by(email="usera@mailbox.test")
    ua.disabled = False
    ua.delete_on = arrow.now()
    Session.commit()
    ua = User.get_by(email="usera@mailbox.test")
    print(f"    user_a.disabled={ua.disabled} delete_on set={ua.delete_on is not None} "
          f"is_active()={ua.is_active()} can_send_or_receive()={ua.can_send_or_receive()}")
    drive("C2 delete_on-user", "resolved_owner_delete_on", user_a.email, alias_a.email, "<f07-c2@sl.local>")

    print("--- C3: RESTORE user_A (disabled=False, delete_on=None) ---")
    ua = User.get_by(email="usera@mailbox.test")
    ua.disabled = False
    ua.delete_on = None
    Session.commit()
    ua = User.get_by(email="usera@mailbox.test")
    print(f"    RESTORED user_a.disabled={ua.disabled} delete_on={ua.delete_on!r} "
          f"is_active()={ua.is_active()} can_send_or_receive()={ua.can_send_or_receive()}")
    print()

    # ============ SECTION D - ADVERSARIAL is_reverse_alias() PREDICATE ============
    print("========== SECTION D: ADVERSARIAL is_reverse_alias() PREDICATE ==========")
    d_before = Session.query(Contact).count()
    adversarial = [
        ("empty", ""),
        ("sql_injection", "' OR '1'='1' --@" + EMAIL_DOMAIN),
        ("sql_drop", "x'; DROP TABLE contact;--@" + EMAIL_DOMAIN),
        ("crlf_injection", "a@" + EMAIL_DOMAIN + "\r\nBcc: attacker@evil.test"),
        ("unicode", "r\u00e9ply\u4e2d\u6587@" + EMAIL_DOMAIN),
        ("overlong_5000", ("a" * 5000) + "@" + EMAIL_DOMAIN),
        ("null_byte", "a\x00b@" + EMAIL_DOMAIN),
    ]
    for name, val in adversarial:
        shown = val if len(val) <= 60 else (val[:57] + "...(%d chars)" % len(val))
        try:
            res = is_reverse_alias(val)
            print(f"  is_reverse_alias({name}={shown!r}) = {res!r}  (returned a bool; no injection)")
        except Exception as e:
            print(f"  is_reverse_alias({name}={shown!r}) RAISED {type(e).__name__}: "
                  f"{str(e)[:80]!r}  (safe: driver rejected input; no injection/execution)")
    d_after = Session.query(Contact).count()
    print(f"  Contact rowcount before={d_before} after={d_after}  (unchanged: predicate performs no writes)")
    print()

    # ======== SECTION E - ADVERSARIAL replace_header_when_reply() ========
    print("========== SECTION E: ADVERSARIAL replace_header_when_reply() (second lookup site) ==========")
    import email as _email
    e_before = Session.query(Contact).count()

    # E1: a REAL reverse-alias in To is replaced by the contact website_email; alias.email is skipped
    msg1 = EmailMessage()
    msg1["To"] = f'"Alias" <{alias_a.email}>, "RA" <{SHARED}>'
    print(f"  E1 To (old) = {msg1['To']!r}")
    try:
        email_handler.replace_header_when_reply(msg1, alias_a, headers.TO)
        print(f"  E1 To (new) = {msg1['To']!r}  (reverse-alias replaced by contact website_email; alias skipped)")
    except NonReverseAliasInReplyPhase as e:
        print(f"  E1 raised NonReverseAliasInReplyPhase({e!r})")

    # E2: the modern EmailMessage policy itself REFUSES a header containing bare CR/LF (defense-in-depth)
    try:
        _m = EmailMessage()
        _m["To"] = "victim@evil.test\r\nBcc: attacker@evil.test"
        print(f"  E2 modern-policy set-time result To = {_m['To']!r}  (unexpected: policy allowed CRLF)")
    except ValueError as e:
        print(f"  E2 modern EmailMessage policy REJECTS CRLF at set-time: ValueError {str(e)[:70]!r} "
              f"(defense-in-depth: the pipeline message object cannot hold an injected header)")

    # E3: a compat32-PARSED message (as the real inbound path parses raw bytes) CAN carry a folded
    #     To header containing CR/LF; replace_header_when_reply strips \r/\n (email_handler.py:355-357)
    raw3 = (b"Message-ID: <f07-e3@sl.local>\r\n"
            b"To: first@evil.test,\r\n\tsecond@evil.test\r\n"
            b"\r\n"
            b"body\r\n")
    msg3 = _email.message_from_bytes(raw3)  # compat32 (lenient) policy, like the real inbound path
    raw_to = [str(h) for h in msg3.get_all("To", [])]
    print(f"  E3 compat32 To (raw) = {raw_to!r}  has_CR={any(chr(13) in h for h in raw_to)} "
          f"has_LF={any(chr(10) in h for h in raw_to)}")
    try:
        email_handler.replace_header_when_reply(msg3, alias_a, headers.TO)
        print(f"  E3 To (new) = {msg3['To']!r}  (no non-reverse address raised)")
    except NonReverseAliasInReplyPhase as e:
        print(f"  E3 raised NonReverseAliasInReplyPhase (non-reverse addr) - CAUGHT; the function got PAST "
              f"the \\r/\\n strip to address parsing, so no CRLF survives into any rewritten header")

    # E4: compat32-parsed adversarial SQL/Unicode non-reverse address -> raises, caught, no write/crash
    raw4 = ("Message-ID: <f07-e4@sl.local>\r\n"
            "To: \"x\"; DROP TABLE contact;--\u4e2d@evil.test\r\n"
            "\r\n"
            "body\r\n").encode("utf-8")
    msg4 = _email.message_from_bytes(raw4)
    print(f"  E4 compat32 To (adversarial) = {[str(h) for h in msg4.get_all('To', [])]!r}")
    try:
        email_handler.replace_header_when_reply(msg4, alias_a, headers.TO)
        print(f"  E4 To (new) = {msg4['To']!r}")
    except NonReverseAliasInReplyPhase as e:
        print(f"  E4 raised NonReverseAliasInReplyPhase - CAUGHT (safe; no crash, no write)")
    except Exception as e:
        print(f"  E4 raised {type(e).__name__}: {str(e)[:80]!r} - CAUGHT (safe; no crash, no write)")

    e_after = Session.query(Contact).count()
    print(f"  Contact rowcount before={e_before} after={e_after}  (unchanged: header rewrite performs no Contact writes)")
    print()
    print("DONE")
```

**`observe_sec.py`** — F-04 security / privacy matrix: adversarial recipients (SQL metacharacter / CRLF / Unicode / overlong / null-byte) driven through the **canonical** `email_handler.handle()`, plus the cross-user E214 disclosure with explicit six-way separation (§8.9–§8.10). The control bytes (NUL, CRLF) are placed on `rcpt_tos` and handed to `handle()` unchanged; only the harness's own summary labels escape them for a clean, ASCII-safe transcript. SHA256 `ea2e056b9f5a5fe3bdbc382310b88131b0492518590ca84e5579544d389e8e0d`. Invocation:
```
docker exec sl_app bash -lc '. /app/venv/bin/activate && set -a && . /tmp/sl_env.sh && set +a && cd /tmp && python observe_sec.py'
```

```python
"""observe_sec.py - F-04 security / privacy matrix (CANONICAL entry email_handler.handle()).

SEC-A: adversarial RECIPIENTS (SQLi-OR / SQLi-DROP / CRLF / Unicode / overlong /
       null-byte) driven through the TOP-LEVEL canonical entry
       email_handler.handle(envelope, msg) [email_handler.py:1945; routing at
       :2195 is_reverse_alias -> handle_reply, else :2208 handle_forward].
       Asserts: no injection (contact/alias/users rowcounts unchanged, the `contact`
       table survives the DROP-TABLE payload) and no uncaught crash beyond a clean
       SMTP status string or a driver-level input rejection.

SEC-B: cross-user DISCLOSURE. A shared reply_email mis-resolves via the unordered
       .first() [app/models.py:82-84] to user_B's contact while user_A replies. The
       E214 owner-alert [email_handler.py:1034 -> handle_unknown_mailbox :1390, send
       to user.email :1393] discloses user_A's mailbox to the wrongly-resolved user_B.
       Explicit six-way separation the finding demands:
         (1) wrong Contact  (2) wrong owner  (3) E214 alert recipient+subject
         (4) successful forward  (5) notification (SentAlert) ownership
         (6) EmailLog ownership.
"""
import os
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI", "") not in _ALLOWED_DSNS:
    sys.stderr.write("REFUSING TO RUN: DB_URI=%r not in disposable allowlist\n" % os.environ.get("DB_URI", ""))
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()
with engine.connect() as _c:
    if _c.execute("select current_database()").scalar() != "test":
        sys.stderr.write("REFUSING TO RUN: current_database() != 'test'\n")
        sys.exit(3)
with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog, User, SentAlert
from app.email import status, headers
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


def row_counts():
    """Fresh short-lived connection so committed rows from handle() are visible.
    Selecting count(*) FROM contact also proves the table still EXISTS (a successful
    DROP TABLE injection would make this raise)."""
    with engine.connect() as c:
        nc = c.execute("select count(*) from contact").scalar()
        na = c.execute("select count(*) from alias").scalar()
        nu = c.execute("select count(*) from users").scalar()
    return (nc, na, nu)


def default_alias_of(user):
    return Session.query(Alias).filter(Alias.user_id == user.id).order_by(Alias.id).first()


def build_msg(mid, from_hdr, to_hdr, subject="sec probe"):
    m = EmailMessage()
    m["From"] = from_hdr
    m["To"] = to_hdr
    m["Message-ID"] = mid
    m["Subject"] = subject
    m["Date"] = "Mon, 13 Jul 2026 00:00:00 +0000"
    m.set_content("security/privacy probe body")
    return m


mail_sender.store_emails_instead_of_sending(True)

with app.app_context():
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()

    # ---- shared seed for SEC-A: one legitimate reverse-alias contact ----
    owner = create_new_user(email="owner@mailbox.test")
    Session.commit()
    alias_o = Alias.create_new_random(owner)
    Session.commit()
    legit_reply = "sec-legit@%s" % EMAIL_DOMAIN
    Contact.create(user_id=alias_o.user_id, alias_id=alias_o.id,
                   website_email="ext@nowhere.net", name="Ext", reply_email=legit_reply)
    Session.commit()

    print("========== SECTION SEC-A: ADVERSARIAL RECIPIENTS through canonical email_handler.handle() ==========")
    print("seed: owner.id=%d owner.email=%r alias_o.id=%d legit reverse-alias=%r"
          % (owner.id, owner.email, alias_o.id, legit_reply))
    base = row_counts()
    print("baseline rowcounts (contact,alias,users) = %r" % (base,))
    print("")

    def _safe_disp(s):
        # Escape control bytes (NUL / CR / LF / tab) so the printed label is a clean,
        # single-line, ASCII-safe string; printable Unicode (e.g. accented / CJK) is kept
        # verbatim. The REAL string -- including the raw control bytes -- is still what is
        # placed on env.rcpt_tos and handed to handle() below, so the adversarial input is
        # exercised unchanged; only this human-readable label is escaped.
        return "".join(
            ch if (ch.isprintable() and ch not in "\r\n\t")
            else ch.encode("unicode_escape").decode("ascii")
            for ch in s
        )

    adversarial = [
        ("sql_or",     "' OR '1'='1' --@%s" % EMAIL_DOMAIN),
        ("sql_drop",   "x'; DROP TABLE contact;--@%s" % EMAIL_DOMAIN),
        ("crlf",       "a@%s\r\nBcc: attacker@evil.test" % EMAIL_DOMAIN),
        ("unicode",    "r\u00e9ply\u4e2d\u6587@%s" % EMAIL_DOMAIN),
        ("overlong",   ("a" * 300) + "@%s" % EMAIL_DOMAIN),
        ("null_byte",  "a\x00b@%s" % EMAIL_DOMAIN),
    ]
    for i, (label, rcpt) in enumerate(adversarial):
        mail_sender.purge_stored_emails()
        before = row_counts()
        shown = _safe_disp(rcpt) if len(rcpt) <= 70 else (_safe_disp(rcpt[:40]) + "...(%d chars)" % len(rcpt))
        mid = "<sec-a%d@sl.local>" % i
        env = Envelope()
        env.mail_from = owner.email
        env.rcpt_tos = [rcpt]
        msg = build_msg(mid, "ext@nowhere.net", legit_reply)
        outcome = None
        try:
            st = email_handler.handle(env, msg)
            outcome = "returned SMTP status = %r" % st
        except Exception as e:
            outcome = "RAISED %s: %r (input rejected before any injection/execution)" % (type(e).__name__, str(e))
        after = row_counts()
        el = EmailLog.get_by(message_id=mid)
        n_out = len(mail_sender.get_stored_emails())
        print("--- [SEC-A%d %s] rcpt_to=%s ---" % (i, label, shown))
        print("    handle() %s" % outcome)
        print("    routed EmailLog for this Message-ID = %s" % ("None" if el is None else "id=%d user_id=%d" % (el.id, el.user_id)))
        print("    outbound stored SendRequest count = %d" % n_out)
        print("    rowcounts before=%r after=%r  UNCHANGED=%s" % (before, after, before == after))
    fin = row_counts()
    print("")
    print("SEC-A final rowcounts (contact,alias,users) = %r  == baseline %r ? %s" % (fin, base, fin == base))
    print("`contact` table survived DROP-TABLE payload (count(*) succeeded above) = True")

    # ---- SEC-B: cross-user disclosure. Reseed B-then-A so .first() resolves user_B. ----
    print("")
    print("========== SECTION SEC-B: CROSS-USER E214 DISCLOSURE (shared reply_email mis-resolves to user_B) ==========")
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()
    user_b = create_new_user(email="userb@mailbox.test")   # created FIRST
    Session.commit()
    user_a = create_new_user(email="usera@mailbox.test")   # created SECOND
    Session.commit()
    alias_b = default_alias_of(user_b)
    alias_a = default_alias_of(user_a)
    SHARED = "dup-disclose@%s" % EMAIL_DOMAIN
    # insert contact_B FIRST (lowest id) so unordered .first() resolves user_B's contact
    contact_b = Contact.create(user_id=alias_b.user_id, alias_id=alias_b.id,
                               website_email="cb@nowhere.net", name="CB", reply_email=SHARED)
    Session.commit()
    contact_a = Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id,
                               website_email="ca@nowhere.net", name="CA", reply_email=SHARED)
    Session.commit()
    resolved = Contact.get_by(reply_email=SHARED)
    print("seed: user_b.id=%d (%r) alias_b.id=%d ; user_a.id=%d (%r) alias_a.id=%d"
          % (user_b.id, user_b.email, alias_b.id, user_a.id, user_a.email, alias_a.id))
    print("      two contacts share reply_email=%r : contact_b.id=%d(user_b) contact_a.id=%d(user_a)"
          % (SHARED, contact_b.id, contact_a.id))
    print("      Contact.get_by(reply_email=SHARED).first() -> id=%d alias_id=%d user_id=%d  (resolves user_B? %s)"
          % (resolved.id, resolved.alias_id, resolved.user_id, resolved.user_id == user_b.id))
    print("      REPLY will be sent by user_A (mail_from=%r) who legitimately owns the OTHER duplicate (contact_a)" % user_a.email)

    # SEC-B1: default control (spoof-check ON) -> E214 owner-alert to WRONG user_B
    print("")
    print("--- [SEC-B1 default control: alias_b.disable_email_spoofing_check=False] ---")
    mail_sender.purge_stored_emails()
    alerts_before = SentAlert.filter().count()
    mid = "<sec-b1@sl.local>"
    env = Envelope()
    env.mail_from = user_a.email
    env.rcpt_tos = [SHARED]
    msg = build_msg(mid, "cb@nowhere.net", SHARED, subject="reply from user_A")
    st = email_handler.handle(env, msg)
    el = EmailLog.get_by(message_id=mid)
    outs = mail_sender.get_stored_emails()
    new_alerts = SentAlert.filter().count() - alerts_before
    print("    (1) WRONG Contact resolved : contact.id=%d alias_id=%d user_id=%d (user_B) -- NOT the sender user_A's contact"
          % (resolved.id, resolved.alias_id, resolved.user_id))
    print("    (2) WRONG owner            : alias.user=user_%d (user_B) but reply mail_from=%r (user_A)"
          % (resolved.user_id, user_a.email))
    print("    (3) handle() status        : %r (==E214? %s)" % (st, st == status.E214))
    print("    (4) successful forward?    : %s" % ("YES" if (el is not None and el.is_reply) else "NO (E214 before EmailLog.create/forward)"))
    print("    (5) EmailLog ownership     : %s" % ("None" if el is None else "id=%d user_id=%d" % (el.id, el.user_id)))
    for sr in outs:
        env_to = sr.envelope_to
        subj = sr.msg[headers.SUBJECT] if sr.msg is not None else None
        print("    (3) E214 alert -> recipient=%r  subject=%r" % (env_to, subj))
    sa = SentAlert.filter().order_by(SentAlert.id.desc()).first()
    if sa is not None:
        print("    (6) notification ownership : SentAlert id=%d user_id=%d alert_type=%r (owned by WRONG user_B)"
              % (sa.id, sa.user_id, sa.alert_type))
    print("    NEW SentAlert rows = %d ; outbound count = %d" % (new_alerts, len(outs)))
    print("    DISCLOSURE: user_A's mailbox %r is exposed in the alert delivered to WRONG user_B (%r)."
          % (user_a.email, user_b.email))

    # SEC-B2: [non-canonical] spoof-disabled fallback -> WRONG-user forward + EmailLog ownership
    print("")
    print("--- [SEC-B2 [non-canonical] fallback: alias_b.disable_email_spoofing_check=True] ---")
    alias_b.disable_email_spoofing_check = True
    Session.commit()
    mail_sender.purge_stored_emails()
    alerts_before = SentAlert.filter().count()
    mid = "<sec-b2@sl.local>"
    env = Envelope()
    env.mail_from = user_a.email
    env.rcpt_tos = [SHARED]
    msg = build_msg(mid, "cb@nowhere.net", SHARED, subject="reply from user_A (spoof off)")
    st = email_handler.handle(env, msg)
    el = EmailLog.get_by(message_id=mid)
    outs = mail_sender.get_stored_emails()
    new_alerts = SentAlert.filter().count() - alerts_before
    print("    (3) handle() status        : %r (==E200? %s)" % (st, st == status.E200))
    print("    (4) successful forward?    : %s" % ("YES" if (el is not None and el.is_reply) else "NO"))
    print("    (5) EmailLog ownership     : %s" % ("None" if el is None else "id=%d contact_id=%d alias_id=%d user_id=%d (WRONG user_B; sender was user_A)" % (el.id, el.contact_id, el.alias_id, el.user_id)))
    for sr in outs:
        print("    (4) forwarded outbound -> envelope_to=%r (contact_b.website_email; user_A's reply reaches user_B's contact)" % sr.envelope_to)
    print("    NEW SentAlert rows = %d ; outbound count = %d" % (new_alerts, len(outs)))
    print("    [non-canonical] label: this path requires disable_email_spoofing_check=True (a per-alias opt-out), mirrors section 10 CASE 2.")

    print("")
    print("DONE")
```

**`observe_second.py`** — the second lookup site `replace_header_when_reply()` (§8.3). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_second.py 2>&1'
```

```python
"""observe_second.py — the SECOND Contact.get_by(reply_email=...) lookup site.

handle_reply() calls replace_header_when_reply() for the TO header
[email_handler.py:1179] and the CC header [email_handler.py:1181]; inside it, a
SECOND lookup Contact.get_by(reply_email=reply_email) runs at email_handler.py:364
(the primary reply lookup is at email_handler.py:986). This harness drives the
canonical handle_reply() with TO and CC headers that each carry a DIFFERENT
reverse-alias, and shows the second site resolving each to its contact and
rewriting the header to that contact's real website_email.
"""
import os
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI", "") not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={os.environ.get('DB_URI','')!r} not in disposable allowlist\n")
    sys.exit(3)

from server import create_app
from app.db import Session, engine

app = create_app()
with engine.connect() as _c:
    if _c.execute("select current_database()").scalar() != "test":
        sys.stderr.write("REFUSING TO RUN: current_database() != 'test'\n")
        sys.exit(3)
with engine.begin() as _c:
    _c.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")

from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog
from app.email import status, headers
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user


def truncate_clean_slate():
    Session.close()
    with engine.begin() as c:
        c.execute(
            "DO $$DECLARE r RECORD; BEGIN "
            "FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname='public' "
            "AND tablename<>'alembic_version') LOOP "
            "EXECUTE 'TRUNCATE TABLE '||quote_ident(r.tablename)||' RESTART IDENTITY CASCADE'; "
            "END LOOP; END$$;"
        )


mail_sender.store_emails_instead_of_sending(True)

with app.app_context():
    truncate_clean_slate()
    add_sl_domains()
    add_proton_partner()
    user = create_new_user(email="second-user@mailbox.test")
    Session.commit()
    alias = Alias.create_new_random(user)
    Session.commit()
    r1 = f"second-r1@{EMAIL_DOMAIN}"
    r2 = f"second-r2@{EMAIL_DOMAIN}"
    c1 = Contact.create(user_id=alias.user_id, alias_id=alias.id, website_email="w1@nowhere.net",
                        name="W1", reply_email=r1)
    c2 = Contact.create(user_id=alias.user_id, alias_id=alias.id, website_email="w2@nowhere.net",
                        name="W2", reply_email=r2)
    Session.commit()

    print("=== second lookup site: replace_header_when_reply() [email_handler.py:364] via TO & CC ===")
    print(f"primary rcpt_to={r1!r} (resolved at :986); TO header={r1!r} -> contact c1.id={c1.id} website='w1@nowhere.net'")
    print(f"CC header={r2!r} -> contact c2.id={c2.id} website='w2@nowhere.net'")
    mid = "<second-0@sl.local>"
    msg = EmailMessage()
    msg["From"] = "w1@nowhere.net"
    msg["To"] = r1
    msg["Cc"] = r2
    msg["Message-ID"] = mid
    msg["Subject"] = "second-lookup"
    msg["Date"] = "Mon, 13 Jul 2026 00:00:00 +0000"
    msg.set_content("body")
    env = Envelope()
    env.mail_from = user.email
    env.rcpt_tos = [r1]
    d, code = email_handler.handle_reply(env, msg, r1)
    Session.commit()
    print(f"handle_reply -> delivered={d} code={code!r} (==E200? {code == status.E200})")
    el = EmailLog.get_by(message_id=mid)
    if el:
        print(f"EmailLog id={el.id} contact_id={el.contact_id} alias_id={el.alias_id} user_id={el.user_id} is_reply={el.is_reply}")
    print(f"rewritten msg[To] = {msg[headers.TO]!r}")
    print(f"rewritten msg[Cc] = {msg[headers.CC]!r}")
    print("DONE")
```

**`contact_schema.sql`** — SQL catalog helper backing the §6.1(b) runtime proof that `reply_email` carries no `UNIQUE` constraint. Unlike the twelve Python harnesses above it is a plain `psql` script (no app bootstrap and no writes): read-only catalog queries against `pg_indexes`, `pg_constraint`, and `pg_index`/`pg_class`/`pg_attribute`. Invocation:
```
docker exec sl_app bash -lc 'su postgres -c "psql -d test -v ON_ERROR_STOP=1 -f /tmp/contact_schema.sql"; echo "psql_exit=$?"'
```

```sql
\pset pager off
\echo '=== indexes on contact (pg_indexes) ==='
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'contact' AND indexdef LIKE '%reply_email%';
\echo '=== constraints on contact (pg_constraint) ==='
SELECT conname, contype, pg_get_constraintdef(oid) AS def FROM pg_constraint WHERE conrelid = 'contact'::regclass ORDER BY conname;
\echo '=== is there ANY unique index/constraint covering reply_email? ==='
SELECT count(*) AS unique_on_reply_email
FROM pg_index i
JOIN pg_class c ON c.oid = i.indrelid
JOIN pg_attribute a ON a.attrelid = c.oid AND a.attnum = ANY (i.indkey)
WHERE c.relname = 'contact' AND i.indisunique AND a.attname = 'reply_email';
```

Its complete, unedited output is the §6.1(b) block above; it is read-only (pure catalog introspection) and data-independent, so the result is fixed by the schema (alembic head `32f25cbf12f6`) and identical on every run.

---
### 11.4 Dependency integrity, available audits & authoritative advisory classification (F-05)

This subsection provides the dependency/security-acceptance evidence for the change: (a) a manifest & lockfile **integrity diff** proving the branch changes **zero** dependencies, (b) the **installed versions** of every package on the reply-resolution path, (c) the **available no-install audits** (`poetry check`, `poetry check --lock`, `pip check`, and the `pip-audit` availability check), (d) the **`npm audit`** of the browser/UI assets, and (e) an **authoritative advisory classification** (range / severity / fix / reply-path relevance / introduced-vs-pre-existing) for every advisory surfaced. Every command below is shown with its **complete, unedited** captured output; the Python and npm audits were re-run and confirmed **byte-identical across two runs**.

**11.4.1 Manifest & lockfile integrity — zero dependency change across the branch.** The task is documentation-only (AAP §0.5.1/§0.5.2): no manifest, lockfile, or product source may change. The diff below proves the entire branch delta versus the source commit `2cd6ee77` is a **single added Markdown file**, and that a diff restricted to the dependency manifests (`pyproject.toml`, `poetry.lock`, `Dockerfile`, `static/package.json`, `static/package-lock.json`) is **empty** — both across the committed branch and in the working tree.

Commands:

```bash
# destination working tree (branch blitzy-3fc9b061-...); source commit 2cd6ee77 is in history
git diff 2cd6ee77 HEAD --name-status
git diff 2cd6ee77 HEAD --stat -- pyproject.toml poetry.lock Dockerfile static/package.json static/package-lock.json
git diff --name-status
git diff --stat -- pyproject.toml poetry.lock Dockerfile static/package.json static/package-lock.json
```

Complete, unedited output:

```
$ git diff 2cd6ee77 HEAD --name-status
A	blitzy/documentation/app_2cd6ee777f8c.md

$ git diff 2cd6ee77 HEAD --stat -- pyproject.toml poetry.lock Dockerfile static/package.json static/package-lock.json
(no output above = zero manifest/lockfile changes across the entire branch)

$ git diff --name-status        # uncommitted working tree
M	blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --stat -- pyproject.toml poetry.lock Dockerfile static/package.json static/package-lock.json   # working tree, manifests only
(no output above = zero manifest/lockfile changes in the working tree)
```

Interpretation: `git diff 2cd6ee77 HEAD --name-status` → `A	blitzy/documentation/app_2cd6ee777f8c.md` (one added file); the manifest-restricted diffs emit nothing (exit 0). **No dependency is added, updated, or removed by this change** — consistent with AAP §0.6 ("No dependency changes"). Therefore every advisory classified in §11.4.5 is, by construction, **pre-existing baseline**, not introduced here.

**11.4.2 Installed versions of the reply-resolution-path packages** (canonical container `sl_app`, Python 3.10.18). These are the versions actually exercised by the harnesses in §11.3 when driving `handle()` / `handle_reply()`:

Command:

```bash
docker exec sl_app bash -lc '. /app/venv/bin/activate; python - <<PY
import importlib.metadata as m
for p in ["aiosmtpd","SQLAlchemy","flanker","dkimpy","pyspf","psycopg2-binary","redis","Flask"]:
    print(f"{p}=={m.version(p)}")
PY'
```

Complete, unedited output:

```
aiosmtpd==1.4.2
SQLAlchemy==1.3.24
flanker==0.9.11
dkimpy==1.0.5
pyspf==2.0.14
psycopg2-binary==2.9.3
redis==4.6.0
Flask==1.1.2
```

`aiosmtpd 1.4.2` is the inbound SMTP server that provides the real reply entry point (AAP §0.6); `SQLAlchemy 1.3.24` is the ORM whose `.first()` semantics are the crux of the wrong-user finding (§4.1, §6.3); `flanker`, `dkimpy`, `pyspf` are the reply-phase parse/gate libraries; `psycopg2-binary` is the PostgreSQL driver (its NUL-byte rejection is observed in §8.9 SEC-A5).

**11.4.3 Available no-install Python audits.** `poetry check` / `poetry check --lock` validate manifest+lock consistency; `pip check` validates the installed dependency graph; `pip-audit` (a CVE scanner) is **not installed** in the image and is labeled UNAVAILABLE rather than silently skipped.

Command:

```bash
docker exec sl_app bash -lc '
cd /app; export PATH="/root/.local/bin:$PATH"; . /app/venv/bin/activate 2>/dev/null
echo "### poetry --version ###"; poetry --version
echo "### poetry check ###"; poetry check; echo "poetry_check_exit=$?"
echo "### poetry check --lock ###"; poetry check --lock; echo "poetry_lock_exit=$?"
echo "### pip check ###"; pip check; echo "pip_check_exit=$?"
echo "### pip-audit availability ###"; command -v pip-audit >/dev/null 2>&1 && echo "pip-audit present" || echo "pip-audit UNAVAILABLE (not installed)"
'
```

Complete, unedited output:

```
### poetry --version ###
Poetry (version 2.1.4)
### poetry check ###
The "poetry.dev-dependencies" section is deprecated and will be removed in a future version. Use "poetry.group.dev.dependencies" instead.
Warning: The "poetry.dev-dependencies" section is deprecated and will be removed in a future version. Use "poetry.group.dev.dependencies" instead.
Warning: [tool.poetry.name] is deprecated. Use [project.name] instead.
Warning: [tool.poetry.version] is set but 'version' is not in [project.dynamic]. If it is static use [project.version]. If it is dynamic, add 'version' to [project.dynamic].
If you want to set the version dynamically via `poetry build --local-version` or you are using a plugin, which sets the version dynamically, you should define the version in [tool.poetry] and add 'version' to [project.dynamic].
Warning: [tool.poetry.description] is deprecated. Use [project.description] instead.
Warning: [tool.poetry.license] is deprecated. Use [project.license] instead.
Warning: [tool.poetry.authors] is deprecated. Use [project.authors] instead.
Warning: [tool.poetry.keywords] is deprecated. Use [project.keywords] instead.
Warning: [tool.poetry.repository] is deprecated. Use [project.urls] instead.
poetry_check_exit=0
### poetry check --lock ###
The "poetry.dev-dependencies" section is deprecated and will be removed in a future version. Use "poetry.group.dev.dependencies" instead.
Warning: The "poetry.dev-dependencies" section is deprecated and will be removed in a future version. Use "poetry.group.dev.dependencies" instead.
Warning: [tool.poetry.name] is deprecated. Use [project.name] instead.
Warning: [tool.poetry.version] is set but 'version' is not in [project.dynamic]. If it is static use [project.version]. If it is dynamic, add 'version' to [project.dynamic].
If you want to set the version dynamically via `poetry build --local-version` or you are using a plugin, which sets the version dynamically, you should define the version in [tool.poetry] and add 'version' to [project.dynamic].
Warning: [tool.poetry.description] is deprecated. Use [project.description] instead.
Warning: [tool.poetry.license] is deprecated. Use [project.license] instead.
Warning: [tool.poetry.authors] is deprecated. Use [project.authors] instead.
Warning: [tool.poetry.keywords] is deprecated. Use [project.keywords] instead.
Warning: [tool.poetry.repository] is deprecated. Use [project.urls] instead.
poetry_lock_exit=0
### pip check ###
No broken requirements found.
pip_check_exit=0
### pip-audit availability ###
pip-audit UNAVAILABLE (not installed)
```

Result: `poetry check` and `poetry check --lock` both exit **0** (PASS). The `Warning:` lines are **cosmetic Poetry-2.1.4 format-deprecation notices** (the image ships a legacy `[tool.poetry]`-style manifest); they are **not** validity failures — the non-zero-warning-but-zero-exit outcome confirms the manifest and lock are internally consistent. `pip check` prints `No broken requirements found.` (exit 0). `pip-audit` is UNAVAILABLE (not installed), so no CVE database scan of the Python tree was run locally; the authoritative advisory review in §11.4.5 covers the reply-path Python packages instead.

**11.4.4 `npm audit` of the browser/UI assets** (`static/`, Node v18.19.0, npm 9.2.0). These packages render the web UI only; **none is on the inbound reply-resolution path**.

Command:

```bash
docker exec sl_app bash -lc '
cd /app/static
node -v; npm -v
echo "### package.json deps ###"; node -e "const p=require(\"./package.json\"); console.log(JSON.stringify({name:p.name,version:p.version,description:p.description,repository:p.repository,keywords:p.keywords,author:p.author,license:p.license,bugs:p.bugs,homepage:p.homepage,dependencies:p.dependencies},null,2))"
echo "### npm audit ###"; npm audit; echo "npm_audit_exit=$?"
'
```

Complete, unedited output:

```
v18.19.0
9.2.0
### package.json deps ###
{
  "name": "simplelogin",
  "version": "1.0.0",
  "description": "Open source email alias solution",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/simple-login/app.git"
  },
  "keywords": [
    "email-alias"
  ],
  "author": "SimpleLogin",
  "license": "MIT",
  "bugs": {
    "url": "https://github.com/simple-login/app/issues"
  },
  "homepage": "https://github.com/simple-login/app#readme",
  "dependencies": {
    "@sentry/browser": "^5.30.0",
    "bootbox": "^5.5.3",
    "font-awesome": "^4.7.0",
    "htmx.org": "^1.6.1",
    "intro.js": "^2.9.3",
    "multiple-select": "^1.5.2",
    "parsleyjs": "^2.9.2",
    "qrious": "^4.0.2",
    "toastr": "^2.1.4",
    "vue": "^2.6.14"
  }
}
### npm audit ###
# npm audit report

@sentry/browser  <7.119.1
Severity: moderate
Sentry SDK Prototype Pollution gadget in JavaScript SDKs - https://github.com/advisories/GHSA-593m-55hh-j8gv
fix available via `npm audit fix --force`
Will install @sentry/browser@10.65.0, which is a breaking change
node_modules/@sentry/browser

bootbox  <=6.0.0
Severity: moderate
Bootbox.js Cross Site Scripting vulnerability - https://github.com/advisories/GHSA-m4ch-4m5f-2gp6
fix available via `npm audit fix --force`
Will install bootbox@6.0.4, which is a breaking change
node_modules/bootbox

vue  2.0.0-alpha.1 - 2.7.16
ReDoS vulnerability in vue package that is exploitable through inefficient regex evaluation in the parseHTML function - https://github.com/advisories/GHSA-5j4c-8p2g-v4jx
fix available via `npm audit fix --force`
Will install vue@3.5.39, which is a breaking change
node_modules/vue

3 vulnerabilities (1 low, 2 moderate)

To address all issues (including breaking changes), run:
  npm audit fix --force
npm_audit_exit=1
```

Reconciliation of the summary line `3 vulnerabilities (1 low, 2 moderate)`: the two entries printing `Severity: moderate` are `@sentry/browser` and `bootbox`; the `vue` entry prints no explicit `Severity:` line in this npm 9.2.0 report format and is the single **low** item (its severity is confirmed low by the GitHub Advisory in §11.4.5). All three fixes are flagged by npm as **breaking** (`npm audit fix --force`), and remediating them is **out of scope** (AAP §0.5.2) — they are recorded here for acceptance, not fixed.

**11.4.5 Authoritative advisory classification** (confirmed against the GitHub Advisory Database / NVD, AAP §0.2.2). "On reply path?" asks whether the package participates in inbound reply resolution; "Introduced?" asks whether *this change* added the advisory (per §11.4.1 the branch changes zero dependencies, so the answer is uniformly **No — pre-existing**).

| # | Package (installed) | Advisory (GHSA / CVE) | Type | Affected range | Fixed in | CVSS / severity | On reply path? | Introduced? |
|---|---------------------|-----------------------|------|----------------|----------|-----------------|----------------|-------------|
| 1 | `aiosmtpd` 1.4.2 | GHSA-pr2m-px7j-xg65 / CVE-2024-27305 | Inbound SMTP smuggling (spoofed sender via non-standard line endings) | `< 1.4.5` | **1.4.5** | 5.3 Medium — `AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:N` | **Yes** — `aiosmtpd` is the inbound SMTP server feeding `handle()` | **No** — pre-existing baseline |
| 2 | `aiosmtpd` 1.4.2 | GHSA-wgjv-9j3q-jhg8 / CVE-2024-34083 | STARTTLS unencrypted command injection (MitM), CWE-349 | `<= 1.4.5` | **1.4.6** | 5.4 Medium — `AV:A/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N` | **Same component** — needs STARTTLS + adjacent MitM | **No** — pre-existing baseline |
| 3 | `@sentry/browser` 5.30.0 (UI) | GHSA-593m-55hh-j8gv (no CVE assigned) | Prototype-pollution gadget in the JS SDK | `< 7.119.1` | **7.119.1** (npm `--force` target 10.65.0, breaking) | Moderate | **No** — browser UI asset | **No** — pre-existing baseline |
| 4 | `bootbox` 5.5.3 (UI) | GHSA-m4ch-4m5f-2gp6 / CVE-2023-46998 | Cross-site scripting via `alert()/confirm()/prompt()` | `3.2` to `6.0` (`<= 6.0.0`) | **6.0.4** (breaking) | Moderate | **No** — browser UI asset | **No** — pre-existing baseline |
| 5 | `vue` 2.6.14 (UI) | GHSA-5j4c-8p2g-v4jx / CVE-2024-9506 | ReDoS in `parseHTML` (regex) | `2.0.0-alpha.1` to `2.7.16` | no non-breaking 2.x fix (npm `--force` target 3.5.39, major/breaking) | Low | **No** — browser UI asset | **No** — pre-existing baseline |

**Classification conclusion (security acceptance):** With **zero** dependency/manifest/lockfile changes proven in §11.4.1, **none** of the five advisories is introduced by this documentation-only change — all are **pre-existing** in the baseline image at commit `2cd6ee77`. Of the five, **only the two `aiosmtpd` advisories touch the reply-resolution path** (aiosmtpd is the inbound SMTP entry point). Both are **Medium** severity and both are **pre-existing**; moreover **neither is exercised by this investigation**, because the harnesses drive `handle()` / `handle_reply()` **in-process** and never open the network SMTP listener where SMTP-smuggling (CVE-2024-27305) or the STARTTLS-injection precondition (CVE-2024-34083) would apply (§2.2, §11.3). The three `npm` advisories are **browser/UI assets under `static/`**, entirely off the reply-resolution path. Remediation of any of these is **explicitly out of scope** (AAP §0.5.2: "Remediation of the identified defect … The objective is to investigate and explain, not to fix"); they are documented here to complete the required changed-vs-pre-existing security classification.

---
### 11.5 Targeted reply/contact regression evidence (F-06)

This subsection substantiates existing-system continuity: that seeding the duplicate cross-user `reply_email` condition and exercising the reply path (the scenarios throughout §7/§10) do **not** break the project's existing reply-handler / contact test coverage. The **exact, clearly-defined targeted scope** is the two complete reply/contact test modules — `tests/test_email_handler.py` (23 nodes) and `tests/test_contact_utils.py` (14 nodes) = **37 nodes**. The run is executed **twice**: once **before** the duplicate-seed scenario ("pre") and once **after** it ("post"), demonstrating the seeded wrong-user condition perturbs none of the existing assertions. Each run's **complete, unedited** output (which, via the project's `pyproject.toml` `-v` addopts, lists every node with its PASSED status) is embedded below with its **count, duration, and exit code**.

**Targeted-not-full-suite limitation (explicit):** this is a **targeted** regression scope of **37 of the 639** tests the suite collects (across **2 of 31** `tests/test_*.py` modules), chosen because these two modules directly cover the reply handler and contact utilities the investigation touches. The **full 639-test suite was not run** here (the canonical harness gates one external-network test behind a timeout per the environment notes); this evidence therefore establishes continuity for the **reply/contact regression surface only**, not the entire suite.

**Exact command:**

```bash
docker exec sl_app bash -lc '
cd /app
. /app/venv/bin/activate
set -a; . /tmp/sl_env.sh; set +a          # CONFIG=tests/test.env, DB_URI, EMAIL_DOMAIN=sl.local, ...
python -m pytest tests/test_email_handler.py tests/test_contact_utils.py \
    -p no:randomly --timeout=60 --timeout-method=signal
'   # pyproject.toml addopts supply -v, hence the per-node PASSED list below
```

**Run 1 — PRE-scenario — complete, unedited output** (`37 passed, 18 warnings in 10.30s`, exit `0`):

```
============================= test session starts ==============================
platform linux -- Python 3.10.18, pytest-8.4.1, pluggy-1.6.0 -- /app/venv/bin/python
cachedir: .pytest_cache
rootdir: /app
configfile: pyproject.toml
plugins: xdist-3.8.0, cov-3.0.0, rerunfailures-15.1, timeout-2.4.0
timeout: 60.0s
timeout method: signal
timeout func_only: False
collecting ... collected 37 items

tests/test_email_handler.py::test_get_mailbox_from_mail_from PASSED      [  2%]
tests/test_email_handler.py::test_should_ignore PASSED                   [  5%]
tests/test_email_handler.py::test_is_automatic_out_of_office PASSED      [  8%]
tests/test_email_handler.py::test_dmarc_forward_quarantine PASSED        [ 10%]
tests/test_email_handler.py::test_gmail_dmarc_softfail PASSED            [ 13%]
tests/test_email_handler.py::test_prevent_5xx_from_spf PASSED            [ 16%]
tests/test_email_handler.py::test_preserve_5xx_with_valid_spf PASSED     [ 18%]
tests/test_email_handler.py::test_preserve_5xx_with_no_header PASSED     [ 21%]
tests/test_email_handler.py::test_dmarc_reply_quarantine[DMARC_POLICY_QUARANTINE] PASSED [ 24%]
tests/test_email_handler.py::test_dmarc_reply_quarantine[DMARC_POLICY_REJECT] PASSED [ 27%]
tests/test_email_handler.py::test_dmarc_reply_quarantine[DMARC_POLICY_SOFTFAIL] PASSED [ 29%]
tests/test_email_handler.py::test_add_alias_to_header_if_needed PASSED   [ 32%]
tests/test_email_handler.py::test_append_alias_to_header_if_needed_existing_to PASSED [ 35%]
tests/test_email_handler.py::test_avoid_add_to_header_already_present PASSED [ 37%]
tests/test_email_handler.py::test_avoid_add_to_header_already_present_in_cc PASSED [ 40%]
tests/test_email_handler.py::test_email_sent_to_noreply PASSED           [ 43%]
tests/test_email_handler.py::test_email_sent_to_noreplies PASSED         [ 45%]
tests/test_email_handler.py::test_references_header PASSED               [ 48%]
tests/test_email_handler.py::test_replace_contacts_and_user_in_reply_phase PASSED [ 51%]
tests/test_email_handler.py::test_send_email_from_non_canonical_address_on_reply PASSED [ 54%]
tests/test_email_handler.py::test_send_email_from_non_canonical_matches_already_existing_user PASSED [ 56%]
tests/test_email_handler.py::test_break_loop_alias_as_mailbox PASSED     [ 59%]
tests/test_email_handler.py::test_preserve_headers PASSED                [ 62%]
tests/test_contact_utils.py::test_create_contact[name-a@b.c-True-True] PASSED [ 64%]
tests/test_contact_utils.py::test_create_contact[None-None-True-True] PASSED [ 67%]
tests/test_contact_utils.py::test_create_contact[None-None-False-True] PASSED [ 70%]
tests/test_contact_utils.py::test_create_contact[None-None-True-False] PASSED [ 72%]
tests/test_contact_utils.py::test_create_contact[None-None-False-False] PASSED [ 75%]
tests/test_contact_utils.py::test_create_contact_email_email_not_allowed PASSED [ 78%]
tests/test_contact_utils.py::test_create_contact_email_email_allowed PASSED [ 81%]
tests/test_contact_utils.py::test_create_contact_name_overrides_email_name PASSED [ 83%]
tests/test_contact_utils.py::test_create_contact_name_taken_from_email PASSED [ 86%]
tests/test_contact_utils.py::test_create_contact_empty_name_is_none PASSED [ 89%]
tests/test_contact_utils.py::test_create_contact_free_user PASSED        [ 91%]
tests/test_contact_utils.py::test_do_not_allow_invalid_email PASSED      [ 94%]
tests/test_contact_utils.py::test_update_name_for_existing PASSED        [ 97%]
tests/test_contact_utils.py::test_update_mail_from_for_existing PASSED   [100%]

=============================== warnings summary ===============================
venv/lib/python3.10/site-packages/pkg_resources/__init__.py:121
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:121: DeprecationWarning: pkg_resources is deprecated as an API
    warnings.warn("pkg_resources is deprecated as an API", DeprecationWarning)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google.logging')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(parent)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('ruamel')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('zope')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

venv/lib/python3.10/site-packages/flask_limiter/errors.py:10
venv/lib/python3.10/site-packages/flask_limiter/errors.py:10
  /app/venv/lib/python3.10/site-packages/flask_limiter/errors.py:10: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    if LooseVersion(werkzeug_version) < LooseVersion("0.9"):  # pragma: no cover

venv/lib/python3.10/site-packages/flask_admin/contrib/__init__.py:2
  /app/venv/lib/python3.10/site-packages/flask_admin/contrib/__init__.py:2: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('flask_admin.contrib')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    __import__('pkg_resources').declare_namespace(__name__)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('flask_admin')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(parent)

venv/lib/python3.10/site-packages/gnupg.py:997
venv/lib/python3.10/site-packages/gnupg.py:997
  /app/venv/lib/python3.10/site-packages/gnupg.py:997: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    rr.setDaemon(True)

venv/lib/python3.10/site-packages/gnupg.py:1004
venv/lib/python3.10/site-packages/gnupg.py:1004
  /app/venv/lib/python3.10/site-packages/gnupg.py:1004: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    dr.setDaemon(True)

venv/lib/python3.10/site-packages/gnupg.py:172
  /app/venv/lib/python3.10/site-packages/gnupg.py:172: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    wr.setDaemon(True)

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
======================= 37 passed, 18 warnings in 10.30s =======================
```

**Scenario executed between the two runs** — seed the duplicate cross-user `reply_email` (two `Contact` rows on aliases owned by *different* users sharing `reply_email='dist-shared@sl.local'`), i.e. the exact wrong-user precondition analyzed in §6.4/§7/§10:

```bash
docker exec sl_app bash -lc '
cd /app; . /app/venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a
python /tmp/observe_dist.py seed ab off      # full source in §11.3
'
```

Its complete, unedited output (persists the duplicate rows; the trailing EXPLAIN confirms the `LIMIT 1 / Seq Scan` lookup path of §4.1/§6.3):

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 04:26:59,148 - SL - DEBUG - 17977 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 04:27:03,631 - SL - INFO - 17977 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-14 04:27:03,634 - SL - INFO - 17977 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-14 04:27:03,635 - SL - INFO - 17977 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-14 04:27:03,637 - SL - DEBUG - 17977 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 04:27:03,919 - SL - INFO - 17977 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 04:27:04,181 - SL - INFO - 17977 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 04:27:04,195 - SL - DEBUG - 17977 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_word591@sl.local
2026-07-14 04:27:04,203 - SL - INFO - 17977 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 04:27:04,212 - SL - DEBUG - 17977 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_word243@sl.local
2026-07-14 04:27:04,219 - SL - INFO - 17977 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== SEED mode order=ab spoof=off (disable_email_spoofing_check=False) ===
shared reply_email='dist-shared@sl.local'  FIXED_MAIL_FROM='usera@mailbox.test'
user_a.id=1 alias_a.id=3  user_b.id=2 alias_b.id=4
insertion #1 -> Contact id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
insertion #2 -> Contact id=2 alias_id=4 user_id=2 website='b-dist@nowhere.net'
--- physical order (ctid) of rows sharing reply_email ---
  ctid=(0,1) id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
  ctid=(0,2) id=2 alias_id=4 user_id=2 website='b-dist@nowhere.net'
--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---
  Limit  (cost=0.14..4.16 rows=1 width=4)
    ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..4.16 rows=1 width=4)
          Index Cond: ((reply_email)::text = 'dist-shared@sl.local'::text)
--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---
  Limit  (cost=0.00..11.00 rows=1 width=4)
    ->  Seq Scan on contact  (cost=0.00..11.00 rows=1 width=4)
          Filter: ((reply_email)::text = 'dist-shared@sl.local'::text)
SEED DONE (persisted; run 'drive' next)
```

**Run 2 — POST-scenario — complete, unedited output** (`37 passed, 18 warnings in 10.42s`, exit `0`):

```
============================= test session starts ==============================
platform linux -- Python 3.10.18, pytest-8.4.1, pluggy-1.6.0 -- /app/venv/bin/python
cachedir: .pytest_cache
rootdir: /app
configfile: pyproject.toml
plugins: xdist-3.8.0, cov-3.0.0, rerunfailures-15.1, timeout-2.4.0
timeout: 60.0s
timeout method: signal
timeout func_only: False
collecting ... collected 37 items

tests/test_email_handler.py::test_get_mailbox_from_mail_from PASSED      [  2%]
tests/test_email_handler.py::test_should_ignore PASSED                   [  5%]
tests/test_email_handler.py::test_is_automatic_out_of_office PASSED      [  8%]
tests/test_email_handler.py::test_dmarc_forward_quarantine PASSED        [ 10%]
tests/test_email_handler.py::test_gmail_dmarc_softfail PASSED            [ 13%]
tests/test_email_handler.py::test_prevent_5xx_from_spf PASSED            [ 16%]
tests/test_email_handler.py::test_preserve_5xx_with_valid_spf PASSED     [ 18%]
tests/test_email_handler.py::test_preserve_5xx_with_no_header PASSED     [ 21%]
tests/test_email_handler.py::test_dmarc_reply_quarantine[DMARC_POLICY_QUARANTINE] PASSED [ 24%]
tests/test_email_handler.py::test_dmarc_reply_quarantine[DMARC_POLICY_REJECT] PASSED [ 27%]
tests/test_email_handler.py::test_dmarc_reply_quarantine[DMARC_POLICY_SOFTFAIL] PASSED [ 29%]
tests/test_email_handler.py::test_add_alias_to_header_if_needed PASSED   [ 32%]
tests/test_email_handler.py::test_append_alias_to_header_if_needed_existing_to PASSED [ 35%]
tests/test_email_handler.py::test_avoid_add_to_header_already_present PASSED [ 37%]
tests/test_email_handler.py::test_avoid_add_to_header_already_present_in_cc PASSED [ 40%]
tests/test_email_handler.py::test_email_sent_to_noreply PASSED           [ 43%]
tests/test_email_handler.py::test_email_sent_to_noreplies PASSED         [ 45%]
tests/test_email_handler.py::test_references_header PASSED               [ 48%]
tests/test_email_handler.py::test_replace_contacts_and_user_in_reply_phase PASSED [ 51%]
tests/test_email_handler.py::test_send_email_from_non_canonical_address_on_reply PASSED [ 54%]
tests/test_email_handler.py::test_send_email_from_non_canonical_matches_already_existing_user PASSED [ 56%]
tests/test_email_handler.py::test_break_loop_alias_as_mailbox PASSED     [ 59%]
tests/test_email_handler.py::test_preserve_headers PASSED                [ 62%]
tests/test_contact_utils.py::test_create_contact[name-a@b.c-True-True] PASSED [ 64%]
tests/test_contact_utils.py::test_create_contact[None-None-True-True] PASSED [ 67%]
tests/test_contact_utils.py::test_create_contact[None-None-False-True] PASSED [ 70%]
tests/test_contact_utils.py::test_create_contact[None-None-True-False] PASSED [ 72%]
tests/test_contact_utils.py::test_create_contact[None-None-False-False] PASSED [ 75%]
tests/test_contact_utils.py::test_create_contact_email_email_not_allowed PASSED [ 78%]
tests/test_contact_utils.py::test_create_contact_email_email_allowed PASSED [ 81%]
tests/test_contact_utils.py::test_create_contact_name_overrides_email_name PASSED [ 83%]
tests/test_contact_utils.py::test_create_contact_name_taken_from_email PASSED [ 86%]
tests/test_contact_utils.py::test_create_contact_empty_name_is_none PASSED [ 89%]
tests/test_contact_utils.py::test_create_contact_free_user PASSED        [ 91%]
tests/test_contact_utils.py::test_do_not_allow_invalid_email PASSED      [ 94%]
tests/test_contact_utils.py::test_update_name_for_existing PASSED        [ 97%]
tests/test_contact_utils.py::test_update_mail_from_for_existing PASSED   [100%]

=============================== warnings summary ===============================
venv/lib/python3.10/site-packages/pkg_resources/__init__.py:121
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:121: DeprecationWarning: pkg_resources is deprecated as an API
    warnings.warn("pkg_resources is deprecated as an API", DeprecationWarning)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google.logging')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('google')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(parent)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('ruamel')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2870: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('zope')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(pkg)

venv/lib/python3.10/site-packages/flask_limiter/errors.py:10
venv/lib/python3.10/site-packages/flask_limiter/errors.py:10
  /app/venv/lib/python3.10/site-packages/flask_limiter/errors.py:10: DeprecationWarning: distutils Version classes are deprecated. Use packaging.version instead.
    if LooseVersion(werkzeug_version) < LooseVersion("0.9"):  # pragma: no cover

venv/lib/python3.10/site-packages/flask_admin/contrib/__init__.py:2
  /app/venv/lib/python3.10/site-packages/flask_admin/contrib/__init__.py:2: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('flask_admin.contrib')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    __import__('pkg_resources').declare_namespace(__name__)

venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349
  /app/venv/lib/python3.10/site-packages/pkg_resources/__init__.py:2349: DeprecationWarning: Deprecated call to `pkg_resources.declare_namespace('flask_admin')`.
  Implementing implicit namespace packages (as specified in PEP 420) is preferred to `pkg_resources.declare_namespace`. See https://setuptools.pypa.io/en/latest/references/keywords.html#keyword-namespace-packages
    declare_namespace(parent)

venv/lib/python3.10/site-packages/gnupg.py:997
venv/lib/python3.10/site-packages/gnupg.py:997
  /app/venv/lib/python3.10/site-packages/gnupg.py:997: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    rr.setDaemon(True)

venv/lib/python3.10/site-packages/gnupg.py:1004
venv/lib/python3.10/site-packages/gnupg.py:1004
  /app/venv/lib/python3.10/site-packages/gnupg.py:1004: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    dr.setDaemon(True)

venv/lib/python3.10/site-packages/gnupg.py:172
  /app/venv/lib/python3.10/site-packages/gnupg.py:172: DeprecationWarning: setDaemon() is deprecated, set the daemon attribute instead
    wr.setDaemon(True)

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
======================= 37 passed, 18 warnings in 10.42s =======================
```

**Stability & interpretation.** Both runs exit `0` with **37 passed, 18 warnings**. A line-by-line diff of Run 1 vs. Run 2 differs in **exactly one line** — the final summary duration (`10.30s` vs `10.42s`); with that timing token neutralized the two outputs are **byte-identical** (invariant SHA256 `dedde263c0ab7ae16aa9df6dd5fc37e5dab5c9589bdfaee5457f667575f02832` for both). Because the **post** run follows the duplicate-`reply_email` seed and still passes all 37 nodes, the seeded wrong-user condition **does not perturb any existing reply/contact assertion** — existing-system continuity is substantiated for this surface.

**Independent corroboration (QA).** The QA agent independently ran a *keyword subset* ("reply / contact / reverse-alias / normalization") of these same modules and observed **26 passed, 18 warnings** in **7.86s** (before scenarios) and **7.68s** (after scenarios), exit `0`. The **identical 18-warnings fingerprint** confirms the same module set; the count differs only because QA's `-k` filter selected 26 of the 37 nodes whereas the scope embedded here is the **complete** two-module set (a superset). Both agree: **all targeted reply/contact tests pass, before and after the scenarios.**

---
## 12. Coverage pass (honest status of every question part and implied condition)

Each row is marked **OBSERVED** (captured at runtime through the canonical entry point), **OBSERVED `[non-canonical]`** (captured but via a bypass/fallback, labeled as such), or **`[inferred]`** (not directly reproduced; grounded in source + official semantics). No row is marked complete beyond what the evidence supports.

| # | Question part / implied condition | Status | Where |
|---|-----------------------------------|--------|-------|
| 1 | Derivation of the reply address (`rcpt_to` → domain gate → normalize) | **OBSERVED** | §3, §4.2 |
| 2 | Contact resolution via `Contact.get_by(reply_email=...)` | **OBSERVED** | §4.2 |
| 3 | Actual extracted/normalized reply-email value reported | **OBSERVED** | §4.2 |
| 4 | Resolved `Contact` reported (`id`, `alias_id`, `user_id`) | **OBSERVED** | §4.2 |
| 5 | Forwarding destination reported (alias, user, mailbox) | **OBSERVED** | §4.2, §5 |
| 6 | Canonical entry point exercised (`handle_reply`) + routing via `handle()` | **OBSERVED** | §4.2, §4.3 |
| 7 | Same reply resolves to different contacts across events? | **OBSERVED** (flips with insertion order; deterministic/stable within a fixed layout, 100/100 ×2, SHA256-verified) | §7 |
| 8 | Reply fails to resolve (no contact) | **OBSERVED** (E502) | §8.1 |
| 9 | Resolves correctly yet forwards to a different user? | **OBSERVED** — default control → **E214 (no forward)**; wrong-user **forward** only under **`[non-canonical]`** spoof-disabled fallback | §7.2, §7.3, §10 |
| 10 | `reply_email` has no DB `UNIQUE` (model + migrations + runtime catalog) | **OBSERVED** | §6.1 |
| 11 | `.first()` has no `ORDER BY`; row is application-unordered | **OBSERVED** (source + EXPLAIN/ctid) | §4.1, §6.3, §7.4 |
| 12 | A *different* plan/layout could return the other row | **`[inferred]`** (grounded in absence of `ORDER BY` + official SQLAlchemy/PostgreSQL semantics; not directly reproduced on the 2-row layout) | §7.4, §9 |
| 13 | Uniqueness assumption: generation-time TOCTOU guard, exact 3 checks | **OBSERVED** | §6.2 |
| 14 | Direct write bypasses the guard (duplicate persists) | **OBSERVED** (duplicate seed) | §6.2, §4.2 |
| 15 | Concurrency/race widening the TOCTOU window | **`[inferred]`** (not reproduced live; grounded in non-atomic read-then-act + no `UNIQUE`) | §6.2 |
| 16 | Wrong contact → wrong alias/user/mailbox/log propagation | **OBSERVED** | §6.4, §7, §10 |
| 17 | Other lookup reuse sites (second header rewrite; routing predicates) | **OBSERVED** (`replace_header_when_reply` canonical; routing via `handle()`) | §8.3, §4.3, §6.5 |
| 18 | Normalization semantics (inbound many-to-one; exact-equality lookup; multi-row needs >1 stored row) | **OBSERVED** | §8.2, §6.6 |
| 19 | E501 (wrong reply domain) | **OBSERVED** | §8.1 |
| 20 | E502 (inactive/soft-deleted user) | **OBSERVED** | §8.1 |
| 21 | E214 (unknown mailbox) + alert to resolved user | **OBSERVED** | §10 |
| 22 | Forward-phase minting (`generate_reply_email` + guard) | **OBSERVED** (source + guard behavior) | §6.7, §6.2 |
| 23 | Official reverse-alias concept (intended unique per sender+alias) | **OBSERVED** (cited) | §9 |
| 24 | Official SQLAlchemy 1.3 / PostgreSQL 15 ordering semantics | **OBSERVED** (cited) | §4.1, §9 |
| 25 | Confirmed external SMTP delivery | **NOT confirmed** — `NOT_SEND_EMAIL=true`; success = application acceptance + enqueue only (explicitly bounded, not claimed) | §2.2, §4.2, §7.3, §10 |
| 26 | Same-unchanged-input, ≥2 runs per condition, every run disclosed | **OBSERVED** | §7 |
| 27 | Temporary harness cleanup (per-file absence proof) | **OBSERVED** | §13 |
| 28 | Manifest/lockfile integrity (zero dependency change across the branch) | **OBSERVED** | §11.4.1 |
| 29 | Available no-install audits (`poetry check`, `poetry check --lock`, `pip check`; `pip-audit` labeled unavailable; `npm audit`) | **OBSERVED** | §11.4.3, §11.4.4 |
| 30 | Advisory research + introduced-vs-pre-existing classification (reply-path `aiosmtpd`; UI-only npm) | **OBSERVED** (cited) | §11.4.5 |
| 31 | Targeted reply/contact regression (37/37 pre+post, byte-stable) + targeted-not-full-suite disclosure | **OBSERVED** | §11.5 |
| 32 | Self-contained clean-room bootstrap (5 setup classes) verified; `KeyFormatError`→`git checkout` fix reproduced live | **OBSERVED** | §2.8 |

**Net (honest, bounded status).** Every part of the *posed question* is answered from runtime observation through the canonical entry point, and all ten QA findings **F-01…F-10 are resolved** with cited, byte-matched evidence (see the per-acceptance-item parity in §12.1 below). Completeness is **bounded, not absolute**: two items remain explicitly `[inferred]` (a different plan/layout returning the other row; a live concurrency race), one is explicitly **not** claimed (confirmed external SMTP delivery under `NOT_SEND_EMAIL=true`), and three items are **out of scope** per AAP §0.5.2 (HTTP/UI; real external SMTP delivery; unrelated remediation). These bounds are restated wherever the related claims appear, not only here; dependency/security acceptance is in rows 28-30 (§11.4) and §11.4.5. **No acceptance item is marked complete beyond what its embedded evidence supports.**

---

### 12.1 AAP / QA acceptance-item parity (independent 53-item matrix, honest status)

The QA review enumerated an independent **53-item** acceptance matrix (its tally: 28 PASS / 22 FAIL / 3 N/A). Every item is reconciled below against this document's embedded evidence. **Legend:** **OBSERVED** = captured at runtime through the canonical path; **RESOLVED (F-0X)** = a gap the QA review flagged, now closed here with cited, byte-matched evidence; **N/A** = explicitly out of scope per AAP §0.5.2. `[inferred]` / `[non-canonical]` / bounded qualifiers are carried from the cited sections.

| # | Acceptance item | Status in this document | Evidence location |
|---|-----------------|-------------------------|-------------------|
| 1 | Canonical image / container | OBSERVED | §2.1, §2.8 |
| 2 | Python / PostgreSQL / Redis / Flask liveness | OBSERVED | §2.1 |
| 3 | Run-first chronology (runtime precedes writing) | OBSERVED | §2, §11 preamble |
| 4 | Reply-domain gate (`EMAIL_DOMAIN` / `SLDomain`) | OBSERVED | §3, §8.1 |
| 5 | Normalization behaviour (`normalize_reply_email`) | OBSERVED | §3, §8.2 |
| 6 | Actual handler-local normalized value | RESOLVED (F-09) | §4.2 |
| 7 | Contact lookup + fields (id / alias_id / user_id) | OBSERVED | §4 |
| 8 | `.first()` with no `ORDER BY` | OBSERVED | §4.1, §6.3, §7.4 |
| 9 | Live constraints / indexes (`reply_email` non-unique) | OBSERVED | §6.1 |
| 10 | Unique-contact baseline (E200 / log / outbound) | OBSERVED | §4.2, §5 |
| 11 | Duplicate cross-user fixture (two distinct owners) | OBSERVED | §4.2, §6.1, §7 |
| 12 | Alias / user / mailbox selection from resolved Contact | OBSERVED | §5, §6.4 |
| 13 | Basic wrong-owner E214 | OBSERVED | §10 |
| 14 | Complete five-sender E214 matrix | RESOLVED (F-07) | §8.4 |
| 15 | Terminal adapter bounding (enqueue; no SMTP claim) | OBSERVED (bounded) | §2.2, §7, §10 |
| 16 | EmailLog identities (present on E200, absent on E214) | OBSERVED | §4, §7, §10 |
| 17 | N>=100 events, two runs, per-event durations | RESOLVED (F-01) | §7 |
| 18 | Identical serialized input (only Message-ID varies) | RESOLVED (F-01) | §7 |
| 19 | Complete distributions (all conditions, both runs) | RESOLVED (F-01) | §7 |
| 20 | Stability wording / no per-call randomness | OBSERVED (+`[inferred]` generality) | §7.4-§7.5 |
| 21 | Genuine synchronized generator race | RESOLVED (F-02) | §6.2.1 |
| 22 | Uniqueness / timing / root-cause chain | RESOLVED (F-02) | §6, §6.2.1 |
| 23 | E501 (wrong reply domain) | OBSERVED | §8.1 |
| 24 | E502 (missing Contact) | OBSERVED | §8.1 |
| 25 | Inactive / soft-deleted owner (E502) | OBSERVED | §8.1 |
| 26 | E504 (disabled) reachable runtime path | RESOLVED (F-07) | §8.5 |
| 27 | Normalization-collision outcomes | OBSERVED | §8.2 |
| 28 | Normal header rewrite (`replace_header_when_reply`) | OBSERVED | §8.3 |
| 29 | Duplicate / malicious header behaviour | RESOLVED (F-04/F-07) | §8.7, §8.9 |
| 30 | Normal reverse-alias predicate dispatch | OBSERVED | §4.3, §8.1 |
| 31 | Adversarial reverse-alias predicate | RESOLVED (F-04/F-07) | §8.6, §8.9 |
| 32 | Targeted reply/contact regressions | RESOLVED (F-06) | §11.5 |
| 33 | Official reverse-alias terminology | OBSERVED (authoritative) | §9 |
| 34 | SQLAlchemy 1.3 / PostgreSQL 15 semantics | OBSERVED (authoritative) | §4.1, §7, §9 |
| 35 | Source citations (`file:line` audit) | OBSERVED | §11.1 |
| 36 | Every claim: exact command + complete output | RESOLVED (F-03/F-08) | §7 (run-2), §2.8 |
| 37 | External / internal links | OBSERVED | References |
| 38 | Hallucination / exactness cross-check | RESOLVED (F-01/F-10) | §7, §12.1 |
| 39 | Security adversarial matrix | RESOLVED (F-04) | §8.9 |
| 40 | Cross-user privacy (E214 alert disclosure) | RESOLVED (F-04) | §8.10 |
| 41 | Secrets / PII hygiene | OBSERVED (synthetic-only; scan clean) | §8.9-§8.10 |
| 42 | Manifest / lockfile diff (zero dependency change) | RESOLVED (F-05) | §11.4.1 |
| 43 | Available no-install audits (poetry / pip / npm) | RESOLVED (F-05) | §11.4.3, §11.4.4 |
| 44 | Advisory research + introduced-vs-pre-existing | RESOLVED (F-05) | §11.4.5 |
| 45 | Clean-room reproducibility (five setup classes) | RESOLVED (F-08) | §2.8 |
| 46 | Harness / DB / container cleanup | OBSERVED | §13 |
| 47 | Read-only repository invariant | OBSERVED | §13 |
| 48 | Prior-finding disposition ledger | OBSERVED (F-01..F-10 closed) | §12.1 |
| 49 | Direct answer + cause chain | OBSERVED | §1, §6, §10 |
| 50 | Complete named coverage (53-item parity) | RESOLVED (F-10) | §12, §12.1 |
| 51 | HTTP API / browser / UI | N/A - out of scope (AAP §0.5.2) | §0.5.2 |
| 52 | Real external SMTP delivery | N/A - bounded, not claimed | §2.2 |
| 53 | Unrelated performance / infra remediation | N/A - out of scope (AAP §0.5.2) | §0.5.2 |

**Parity result:** of the 53 independent acceptance items, **30 are OBSERVED**, **20 the QA review flagged are now RESOLVED** here with cited, byte-matched evidence (F-01…F-10 plus the secrets/PII and prior-finding-ledger acceptance items), and **3 are N/A (out of scope)**. This reconciles the independent **28 PASS / 22 FAIL / 3 N/A** tally to **50 OBSERVED-or-RESOLVED / 3 N/A**, with the residual `[inferred]` and bounded items restated in place (they are not counted as unqualified passes).

---
## 13. Repository invariant & cleanup proof

**Cleanup — explicit per-file absence proof.** Every temporary harness lived in `/tmp` (host `/tmp/harness/`, container `/tmp/`), outside the repository. After capturing evidence they were removed; a bounded `test ! -e` per named script confirms absence (the command prints `ABSENT` for each and exits 0), alongside a `find` that returns nothing:

**Command:**

```
for f in _bootstrap observe_reply observe_handle observe_wronguser observe_dist \
         observe_edge observe_norm observe_avail observe_race observe_f07 observe_sec observe_second; do
  test ! -e "/tmp/harness/$f.py" && echo "ABSENT(host): /tmp/harness/$f.py" || echo "PRESENT(host): /tmp/harness/$f.py"
  docker exec sl_app bash -lc "test ! -e /tmp/$f.py && echo 'ABSENT(container): /tmp/$f.py' || echo 'PRESENT(container): /tmp/$f.py'"
done
echo "--- find (host) ---"; find /tmp/harness -maxdepth 1 \( -name 'observe_*.py' -o -name '_bootstrap.py' \) 2>&1 | sort
echo "--- ls (container) ---"; docker exec sl_app bash -lc "ls /tmp/observe_*.py /tmp/_bootstrap.py 2>&1 | sort"
```

**Output (complete, unedited; captured immediately after the harnesses were deleted):**

```
ABSENT(host): /tmp/harness/_bootstrap.py
ABSENT(container): /tmp/_bootstrap.py
ABSENT(host): /tmp/harness/observe_reply.py
ABSENT(container): /tmp/observe_reply.py
ABSENT(host): /tmp/harness/observe_handle.py
ABSENT(container): /tmp/observe_handle.py
ABSENT(host): /tmp/harness/observe_wronguser.py
ABSENT(container): /tmp/observe_wronguser.py
ABSENT(host): /tmp/harness/observe_dist.py
ABSENT(container): /tmp/observe_dist.py
ABSENT(host): /tmp/harness/observe_edge.py
ABSENT(container): /tmp/observe_edge.py
ABSENT(host): /tmp/harness/observe_norm.py
ABSENT(container): /tmp/observe_norm.py
ABSENT(host): /tmp/harness/observe_avail.py
ABSENT(container): /tmp/observe_avail.py
ABSENT(host): /tmp/harness/observe_race.py
ABSENT(container): /tmp/observe_race.py
ABSENT(host): /tmp/harness/observe_f07.py
ABSENT(container): /tmp/observe_f07.py
ABSENT(host): /tmp/harness/observe_sec.py
ABSENT(container): /tmp/observe_sec.py
ABSENT(host): /tmp/harness/observe_second.py
ABSENT(container): /tmp/observe_second.py
--- find (host) ---
--- ls (container) ---
ls: cannot access '/tmp/_bootstrap.py': No such file or directory
ls: cannot access '/tmp/observe_*.py': No such file or directory
```

The thirteenth temporary script — the `psql` helper `contact_schema.sql` (source in §11.3) that produced the §6.1(b) catalog proof — was likewise removed after use; its absence is confirmed the same way (`.sql`, not `.py`, so it is checked separately):

**Command:**

```
f=contact_schema.sql
test ! -e "/tmp/harness/$f" && echo "ABSENT(host): /tmp/harness/$f" || echo "PRESENT(host): /tmp/harness/$f"
docker exec sl_app bash -lc "test ! -e /tmp/$f && echo 'ABSENT(container): /tmp/$f' || echo 'PRESENT(container): /tmp/$f'"
echo "--- ls (container) ---"; docker exec sl_app bash -lc "ls /tmp/contact_schema.sql 2>&1"
```

**Output (complete, unedited; captured immediately after the helper was deleted):**

```
ABSENT(host): /tmp/harness/contact_schema.sql
ABSENT(container): /tmp/contact_schema.sql
--- ls (container) ---
ls: cannot access '/tmp/contact_schema.sql': No such file or directory
```

**Repository integrity — scoped to the destination repository.** All integrity claims are scoped to the **destination** repository (branch `blitzy-3fc9b061-bac4-42a9-b984-8eaab62d81e6`, §2.3); the canonical container's `/app` is a separate, detached, setup-dirty checkout (§2.3) and is used only as the read-only runtime. In the destination repository, the sole change is the creation of this one Markdown file:

**Command + output (complete, unedited):**

```
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
$ git diff --check; echo "exit=$?"
exit=0
```

No product, source, model, migration, test, configuration, or manifest file was modified; no dependency or lockfile entry was added, updated, or removed; and no defect remediation was performed. The investigated condition (a `reply_email` with no `UNIQUE` constraint resolved by an unordered `.first()`) is documented, not fixed — by design and per scope.

**Authoring-time working-tree snapshot.** The `git status --porcelain` output above shows this one file as `` M `` (modified, not yet staged) because it was captured **while the deliverable was being authored**, before its own commit. Once the file is committed, `git status --porcelain` for this path is **empty** (a clean working tree); the `` M `` therefore reflects the in-progress authoring state, not a persistent modification. The invariant is identical either way: across **all** commits that have ever touched this path (`864d5041` → `1639aadf` → `bcc20d65` → the commit carrying these edits), the **only** changed file is `blitzy/documentation/app_2cd6ee777f8c.md` (each verified name-only), so the integrity claim holds both before and after commit.

---

## References (resolvable citations)

- SimpleLogin Docs — Reverse alias: https://simplelogin.io/docs/getting-started/reverse-alias/
- SimpleLogin Docs — Send emails from your alias: https://simplelogin.io/docs/getting-started/send-email/
- SimpleLogin FAQ: https://simplelogin.io/faq/
- SQLAlchemy 1.3 — Query API (`Query.first()`): https://docs.sqlalchemy.org/en/13/orm/query.html
- SQLAlchemy 1.3 FAQ — Why is ORDER BY required with LIMIT: https://docs.sqlalchemy.org/en/13/faq/ormconfiguration.html#why-is-order-by-required-with-limit-especially-with-subqueryload
- PostgreSQL 15 — §7.5 Sorting Rows (ORDER BY): https://www.postgresql.org/docs/15/queries-order.html
- PostgreSQL 15 — SELECT: https://www.postgresql.org/docs/15/sql-select.html
