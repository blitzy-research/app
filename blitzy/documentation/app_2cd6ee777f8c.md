# SimpleLogin Reply-Resolution Investigation — How an Inbound Reply Resolves to a Contact and Forwarding Destination (and Why It Can Forward to the Wrong User)

> **Scope.** Read-only, run-first root-cause investigation. This Markdown file is the *only* persistent artifact produced. No product, source, model, migration, test, configuration, or manifest file was modified, and **no remediation** (no `UNIQUE` constraint, no `ORDER BY`, no atomic generator, no lock/retry) was performed. All temporary observation scripts live **outside** the repository (in `/tmp`) and were removed afterward, with explicit absence proof in §13.
>
> **Methodology.** Every behavioral claim is backed by the *actual, complete, unedited* output of a command that exercised the **canonical** inbound reply path — `email_handler.handle_reply(envelope, msg, rcpt_to)` (`email_handler.py:966`), and, for the routing claim, the real routing hub `email_handler.handle(envelope, msg)` (`email_handler.py:1945`) — inside the canonical Docker runtime (Python 3.10 + PostgreSQL 15 + Redis). Factual claims are grounded in a specific `file:line`. Values obtained by bypassing the entry point (direct ORM/helper calls) are labeled **`[non-canonical]`**; the per-alias spoof-check-disabled path is labeled **`[non-canonical]` fallback**; statements not directly observed are labeled **`[inferred]`**.
>
> **Evidence fidelity note.** Each command and its output are reproduced from the exact files captured at runtime (saved with combined `stdout`+`stderr`, i.e. `2>&1` — no stream is suppressed). The **only** transformation applied when embedding them here is the removal of *trailing* whitespace at line ends (present solely in `psql`'s column-padding) and the normalization of the final newline, so that `git diff --check` reports no warnings (§7.4, §13). No value, identifier, count, log line, ordering, or structural content was altered. Every behavioral section shows the command that produced the output and was confirmed stable across **≥2 runs** (run 1 is shown; run 2 is byte-identical modulo the intentionally varied Message-ID prefix / random reverse-alias token / timestamps, as noted per section).

---
## 1. Lead Answer (Executive Summary)

**How an inbound reply resolves to a contact and a forwarding destination.** When a user replies to a *reverse-alias*, the inbound SMTP recipient (`rcpt_to`) is taken verbatim as the reply address: `reply_email = rcpt_to` (`email_handler.py:972`). It is gated against the service domain — `if not reply_email.endswith(EMAIL_DOMAIN)` (`email_handler.py:977`), falling back to an `SLDomain` lookup (`email_handler.py:978`) and returning `status.E501` if neither matches (`email_handler.py:980-981`). It is then normalized: `reply_email = normalize_reply_email(reply_email)` (`email_handler.py:984`). The normalized value resolves a **single** `Contact`: `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:986`). The **forwarding destination is derived entirely from that resolved contact**: `alias = contact.alias` (`email_handler.py:994`), `user = alias.user` (`email_handler.py:1004`), and the sender mailbox is chosen by `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:1019`). On the authorized path the resolution is persisted as `EmailLog.create(contact_id=contact.id, alias_id=contact.alias_id, is_reply=True, user_id=contact.user_id, mailbox_id=mailbox.id, ...)` (`email_handler.py:1042-1050`).

**Why it *can* forward to the wrong user.** The lookup `Contact.get_by(reply_email=...)` delegates to the shared `ModelMixin.get_by()`, whose entire body is `return Session.query(cls).filter_by(**kw).first()` — a `.first()` with **no `ORDER BY`** (`app/models.py:82-84`). The `reply_email` column carries **no `UNIQUE` constraint**: the only unique constraint on `Contact` is `uq_contact` on `(alias_id, website_email)` (`app/models.py:1875`); `reply_email` is merely `index=True` (`app/models.py:1899`), and the migration that created its index sets `unique=False` (`migrations/versions/2021_071310_78403c7b8089_.py:22`). Generation-time uniqueness is only a best-effort, non-atomic time-of-check-to-time-of-use (TOCTOU) guard, `available_sl_email()` (`app/models.py:1425-1432`), which non-generator write paths (a direct `Contact.create(...)`) bypass entirely. Consequently, **two `Contact` rows owned by different users can share one `reply_email`**, and on such a multi-row match `.first()` returns one row **without any application-specified order**. Because the alias and user are derived from that row, the reply is routed to — and, on the authorized path, logged against — **whichever contact the database returned**, not necessarily the alias owner whose mailbox actually sent the reply.

**What the runtime evidence actually shows (and the important distinction the default access control enforces).** Seeding two contacts that share one `reply_email` on aliases owned by different users, then driving the *identical* canonical input against those same persisted rows, the resolved contact — and therefore the alias, user, and mailbox derived from it — is determined by which row `.first()` returns, which in turn tracks the contacts' **insertion order** under the active query plan (§7):

- Insertion order **A-then-B** → **20/20** events (× 2 runs) resolved to the first-inserted contact, `user_A` (the correct owner of the sending mailbox). Result: **E200**, `EmailLog.user_id = 1`.
- Insertion order **B-then-A**, with the **identical** `mail_from` (user_A's mailbox) and `rcpt_to` → `.first()` resolves `user_B`'s contact — **a different user than the sending mailbox owner**. What happens next depends on the alias's spoof-check setting:
  - **Default control (`disable_email_spoofing_check = False`, the out-of-the-box behavior)** → `get_mailbox_from_mail_from()` returns `None` (user_A's mailbox is not authorized for user_B's alias), so `handle_reply()` calls `handle_unknown_mailbox()` and returns **`status.E214`** (`email_handler.py:1032-1034`). The reply is **rejected before any `EmailLog` is created and before any forward**; the only outbound message is an *alert* to `user_B` warning that someone tried to use their alias. This is an **access-control boundary**, not a wrong-user delivery. Observed **20/20** (× 2 runs): `code = '250 SL E214 Unauthorized for using reverse alias'`, `EmailLog = None`.
  - **`[non-canonical]` fallback (`disable_email_spoofing_check = True`, a non-default per-alias flag)** → the spoof check is skipped (`email_handler.py:1023`), the sender mailbox falls back to `alias.mailbox`, and the reply is **selected, logged, and a send request is enqueued under the wrong user**: **E200**, `EmailLog.user_id = 2 (user_B)`, external target `b@nowhere.net`. Observed **20/20** (× 2 runs). This is the condition under which a reply is actually *forwarded* under the wrong user.

**Two clarifications the evidence makes precise.** (1) *Application acceptance is not confirmed external delivery.* The harness runs with `NOT_SEND_EMAIL=true`, so `mail_sender.send()` logs and returns success **without** calling the SMTP transport (`app/mail_sender.py:130-136`); an observed `'250 Message accepted for delivery'` and a stored `SendRequest` prove *selection, logging, and enqueue*, not a confirmed outbound SMTP hand-off. (2) *The resolved row is not attributable to a specified ordering.* The application specifies no `ORDER BY`; the row the database returns depends on the active plan and physical layout (§7.4). Within each fixed layout the result was **stable** (20/20 across 2 runs), not random per call; that a *different* plan or layout could return the other row is **`[inferred]`** from the absence of an `ORDER BY` plus the official SQLAlchemy/PostgreSQL semantics cited in §4.1 and §9.

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
- `Alias.create_new_random(user)` creates one alias per user (the boot logs show the generated alias emails, e.g. `generate email test_test113@sl.local`).
- Two `Contact` rows are created on aliases owned by **different** users but with the **same** `reply_email`, via direct `Contact.create(...)`. This is accepted because there is no `UNIQUE` constraint on `reply_email` (proof in §6.1) and because `Contact.create` guards only that `website_email` is not itself a reverse-alias — it does **not** check `reply_email` uniqueness.

### 2.6 Observation classes: canonical, `[non-canonical]`, `[inferred]`

- **Canonical** observations drive the real inbound reply entry point `email_handler.handle_reply(envelope, msg, rcpt_to)` (`email_handler.py:966`); the routing claim is additionally proven by invoking the real hub `email_handler.handle(envelope, msg)` (`email_handler.py:1945`), which detects the reverse-alias and dispatches to `handle_reply()` (§7 is driven through `handle_reply`; §3–§5 use `handle_reply`, and the routing proof in §4.3 uses `handle()`).
- **`[non-canonical]`** observations — a direct `Contact.get_by(reply_email=...)`, a direct `is_reverse_alias(...)` predicate call, or the per-alias **spoof-check-disabled fallback** — are explicitly labeled `[non-canonical]` wherever they appear. They corroborate the underlying `.first()` mechanism or the selection/logging behavior but do not, by themselves, represent default canonical delivery.
- **`[inferred]`** statements (e.g., "a different query plan or physical layout could return the other row") are labeled as such and grounded in the official semantics cited in §4.1 and §9.
- **Scale.** The cross-event experiment drives the identical input **N=20** per run and repeats **each condition across 2 independent drive runs** against the **same** persisted rows (§7). Every seed and every drive run performed against the clean-slate database is disclosed in §7 — none is omitted.

### 2.7 Temporary harness scripts (ten), safety, and the reproducible-command convention

Ten throwaway scripts were used — nine Python harnesses plus one `psql` SQL helper (`contact_schema.sql`). They live **outside** the repository at `/tmp/harness/` on the host (copied into the container's `/tmp/`), and all were removed afterward with per-file absence proof (§13). Their **full source is reproduced in §11.3** so results remain reproducible after deletion:

- `_bootstrap.py` — shared fail-fast bootstrap (documented below).
- `observe_reply.py` — canonical happy-path single call (§3–§5).
- `observe_wronguser.py` — CRITICAL default-control vs `[non-canonical]` fallback (§10).
- `observe_handle.py` — canonical routing through `handle()` (§4.3).
- `observe_dist.py` — same-unchanged-input distribution, seed-once/drive-many (§7).
- `observe_edge.py` — E501 / E502(no-contact) / E502(inactive-user) / `is_reverse_alias` (§8.1).
- `observe_norm.py` — inbound normalization / exact-equality lookup (§8.2).
- `observe_avail.py` — exact `available_sl_email` body, call sites, guard bypass (§6.2).
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

The observed extracted/normalized value for the canonical happy-path call appears inline in §4.2 (fields `rcpt_to (raw)`, `endswith(EMAIL_DOMAIN)`, `normalize_reply_email(...)`).

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

The harness `observe_reply.py` seeds two contacts sharing one `reply_email` on aliases owned by different users, proves the duplicate persisted, prints the reply-address derivation values, then drives the **canonical** `email_handler.handle_reply(envelope, msg, shared_reply)` **once** with `mail_from = user_A`'s mailbox (the owner of the first-inserted contact's alias). Direct ORM/predicate corroboration is explicitly `[non-canonical]`.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_reply.py 2>&1'
```

**Output (complete, unedited; run 1 of 2 — run 2 is byte-identical modulo per-run-varying fields only: logger timestamps/PID and the randomly-generated alias local-parts / reverse-alias token in the stored `SendRequest`):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:24:49,097 - SL - DEBUG - 5735 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 18:24:53,376 - SL - INFO - 5735 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-13 18:24:53,379 - SL - INFO - 5735 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-13 18:24:53,381 - SL - INFO - 5735 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-13 18:24:53,382 - SL - DEBUG - 5735 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:24:53,665 - SL - INFO - 5735 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 18:24:53,926 - SL - INFO - 5735 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 18:24:53,939 - SL - DEBUG - 5735 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_test113@sl.local
2026-07-13 18:24:53,946 - SL - INFO - 5735 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 18:24:53,955 - SL - DEBUG - 5735 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_test759@sl.local
2026-07-13 18:24:53,962 - SL - INFO - 5735 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== SEED (insertion order A-then-B) ===
EMAIL_DOMAIN='sl.local'  NOT_SEND_EMAIL=True
shared_reply='dup-reply-core@sl.local'
user_a.id=1 email='usera@mailbox.test' alias_a.id=3 alias_a.email='test_test113@sl.local'
user_b.id=2 email='userb@mailbox.test' alias_b.id=4 alias_b.email='test_test759@sl.local'
contact_a.id=1 (alias_a,user_a)  contact_b.id=2 (alias_b,user_b)
=== PROVE DUPLICATE PERSISTED (no UNIQUE constraint blocked it) ===
rows sharing reply_email='dup-reply-core@sl.local': count=2
  Contact id=1 alias_id=3 user_id=1 website_email='a@nowhere.net'
  Contact id=2 alias_id=4 user_id=2 website_email='b@nowhere.net'
=== reply-address derivation values (email_handler.py:972,977,984) ===
rcpt_to (raw)              = 'dup-reply-core@sl.local'
endswith(EMAIL_DOMAIN)     = True
normalize_reply_email(...) = 'dup-reply-core@sl.local'
=== [non-canonical] routing predicate + direct lookup (BYPASS the entry point) ===
[non-canonical] is_reverse_alias('dup-reply-core@sl.local') = True
[non-canonical] Contact.get_by(reply_email=...) -> id=1 alias_id=3 user_id=1
=== CANONICAL CALL: email_handler.handle_reply(envelope, msg, shared_reply) ===
envelope.mail_from='usera@mailbox.test' envelope.rcpt_tos=['dup-reply-core@sl.local'] Message-ID='<obs-core-0@sl.local>'
2026-07-13 18:24:53,982 - SL - INFO - 5735 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 18:24:53,988 - SL - DEBUG - 5735 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 a@nowhere.net 3>, <User 1 Test User usera@mailbox.test>, <Mailbox 1 usera@mailbox.test>
2026-07-13 18:24:53,993 - SL - DEBUG - 5735 - "/app/email_handler.py:1171" - handle_reply() -  - From header is test_test113@sl.local
2026-07-13 18:24:53,994 - SL - DEBUG - 5735 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the To header. Old value test_test113@sl.local
2026-07-13 18:24:53,994 - SL - DEBUG - 5735 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 18:24:53,996 - SL - DEBUG - 5735 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178396709399.5735.1120116079739055429.1@sl.local>
2026-07-13 18:24:54,000 - SL - WARNING - 5735 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-13 18:24:54,002 - SL - DEBUG - 5735 - "/app/email_handler.py:1212" - handle_reply() -  - send email from test_test113@sl.local to a@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 18:24:54,006 - SL - DEBUG - 5735 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply subject', from 'test_test113@sl.local' to 'None'
RESULT delivered=True code='250 Message accepted for delivery'
code==status.E200? True  ==E214? False  ==E502? False
PERSISTED EmailLog (message_id='<obs-core-0@sl.local>'): id=1 contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  -> forwarding user_id=1 (user_a.id=1, user_b.id=2)
stored SendRequest count = 1
  SendRequest envelope_from='sl.lmysyibrfqqdemzygi4dmnc5.x4uis64jqu3wi@sl.local' envelope_to='a@nowhere.net' msg[From]='test_test113@sl.local' msg[To]=None
NOTE: NOT_SEND_EMAIL=True => mail_sender.send() logs and returns True WITHOUT calling _send_to_smtp; no external SMTP delivery is confirmed (app/mail_sender.py:130-136).
NEW SentAlert rows during call: 0
DONE
```

**Reading the output (each value grounded):**

- **Duplicate persisted (no `UNIQUE` blocked it):** `rows sharing reply_email='dup-reply-core@sl.local': count=2` — `Contact id=1` (`alias_id=3`, `user_id=1`, `a@nowhere.net`) and `Contact id=2` (`alias_id=4`, `user_id=2`, `b@nowhere.net`). This is the real data condition the whole investigation rests on, and it is accepted by the schema (§6.1).
- **Reply-address derivation (`email_handler.py:972,977,984`):** `rcpt_to (raw) = 'dup-reply-core@sl.local'`, `endswith(EMAIL_DOMAIN) = True`, `normalize_reply_email(...) = 'dup-reply-core@sl.local'` (unchanged — every character is already in `_ALLOWED_CHARS`; the collision behavior for other inputs is in §8.2).
- **`[non-canonical]` corroboration (bypasses the entry point, labeled):** `is_reverse_alias('dup-reply-core@sl.local') = True`; `Contact.get_by(reply_email=...) -> id=1 alias_id=3 user_id=1`.
- **Canonical resolution:** the real `handle_reply()` call resolves `Contact 1` and logs at `email_handler.py:1051`: `Create <EmailLog 1> for <Contact 1 a@nowhere.net 3>, <User 1 ... usera@mailbox.test>, <Mailbox 1 usera@mailbox.test>`.
- **Persisted `EmailLog`, correlated by exact Message-ID** (`<obs-core-0@sl.local>`): `id=1 contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True` → forwarding `user_id=1` (the correct owner).
- **Application acceptance vs. delivery:** `delivered=True code='250 Message accepted for delivery'` (`==status.E200? True`). One `SendRequest` was stored: `envelope_from='sl.l...@sl.local' envelope_to='a@nowhere.net' msg[From]='test_test113@sl.local' msg[To]=None`. The explicit `NOTE` records that with `NOT_SEND_EMAIL=True`, `mail_sender.send()` logs and returns `True` **without** calling `_send_to_smtp` (`app/mail_sender.py:130-136`) — so no external SMTP delivery is confirmed. `NEW SentAlert rows during call: 0`.

**Two diagnostics explained (so the log is not misread):**

- **`missing date header, add one` (`email_handler.py:1206`, `WARNING`).** The synthetic reply message carries no `Date:` header, so `handle_reply()` adds one before sending. It is an expected normalization for a hand-constructed message, not an error.
- **`send email ... to 'None'` (`app/mail_sender.py:131`) while `envelope_to='a@nowhere.net'`.** These refer to *different* things. The mail-sender log prints the **message header** `To:` (`msg[headers.TO]`), which is `None` here because `replace_header_when_reply()` deleted the reverse-alias `To` header (`email_handler.py:383` — `delete the To header`) since the original `To` was the reverse-alias itself. The **SMTP envelope recipient** is separate and is the real contact address: `SendRequest.envelope_to='a@nowhere.net'` (`= contact.website_email`). In other words, the *message header* being `None` and the *envelope recipient* being the contact are simultaneously correct and non-contradictory.

### 4.3 Canonical routing proof — `email_handler.handle()` dispatches to `handle_reply()`

To prove the routing claim rather than merely corroborate it from source, `observe_handle.py` invokes the **real routing hub** `email_handler.handle(envelope, msg)` (not `handle_reply` directly) with a reverse-alias recipient, and observes the hub detect the reverse-alias and dispatch into the reply phase.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_handle.py 2>&1'
```

**Output (complete, unedited; run 1 of 2 — run 2 byte-identical modulo per-run-varying fields only: logger timestamps/PID, the randomly-generated alias local-parts, and the generated `sl_message_id`):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:28:49,954 - SL - DEBUG - 5885 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 18:28:54,270 - SL - INFO - 5885 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-13 18:28:54,273 - SL - INFO - 5885 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-13 18:28:54,275 - SL - INFO - 5885 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
2026-07-13 18:28:54,276 - SL - DEBUG - 5885 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 18:28:54,565 - SL - INFO - 5885 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 18:28:54,582 - SL - DEBUG - 5885 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_list813@sl.local
2026-07-13 18:28:54,589 - SL - INFO - 5885 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== CANONICAL ROUTING: email_handler.handle(envelope, msg) (NOT handle_reply directly) ===
reply_email(reverse-alias)='route-check@sl.local'  mail_from='usera@mailbox.test'  Message-ID='<route-0@sl.local>'
2026-07-13 18:28:54,598 - SL - DEBUG - 5885 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-13 18:28:54,599 - SL - DEBUG - 5885 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:usera@mailbox.test, rcpt_tos:['route-check@sl.local'], header_from:a@nowhere.net, header_to:route-check@sl.local, cc:None, reply-to:None, message_id:<route-0@sl.local>, client_ip:None, headers:[('From', 'a@nowhere.net'), ('To', 'route-check@sl.local'), ('Message-ID', '<route-0@sl.local>'), ('Subject', 'reply via handle()'), ('Date', 'Mon, 13 Jul 2026 00:00:00 +0000'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-13 18:28:54,603 - SL - DEBUG - 5885 - "/app/email_handler.py:2196" - handle() -  - Reply phase usera@mailbox.test(a@nowhere.net) -> route-check@sl.local
2026-07-13 18:28:54,605 - SL - INFO - 5885 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 18:28:54,610 - SL - DEBUG - 5885 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 1> for <Contact 1 a@nowhere.net 2>, <User 1 Test User usera@mailbox.test>, <Mailbox 1 usera@mailbox.test>
2026-07-13 18:28:54,617 - SL - DEBUG - 5885 - "/app/email_handler.py:1171" - handle_reply() -  - From header is list_list813@sl.local
2026-07-13 18:28:54,618 - SL - DEBUG - 5885 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: route-check@sl.local, new: A <a@nowhere.net>
2026-07-13 18:28:54,619 - SL - DEBUG - 5885 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 18:28:54,621 - SL - DEBUG - 5885 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178396733462.5885.12396735564806354028.1@sl.local>
2026-07-13 18:28:54,627 - SL - DEBUG - 5885 - "/app/email_handler.py:1212" - handle_reply() -  - send email from list_list813@sl.local to a@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 18:28:54,631 - SL - DEBUG - 5885 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply via handle()', from 'list_list813@sl.local' to 'A <a@nowhere.net>'
handle() returned SMTP status = '250 Message accepted for delivery'  (==E200? True)
handle() routed to handle_reply -> EmailLog id=1 contact_id=1 alias_id=2 user_id=1 is_reply=True
DONE
```

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

**Observed forwarding destination (canonical happy path).** The values are in the §4.2 output block: `alias_id=3` (`test_test113@sl.local`), `user_id=1`, `mailbox_id=1` (`usera@mailbox.test`), and the rewritten `From` header `From header is test_test113@sl.local` (`email_handler.py:1171`). The `From` is rewritten to the alias so the recipient sees the alias, not the mailbox — matching the SimpleLogin reverse-alias contract (§9). The wrong-user variants of this selection (E214 default vs. `[non-canonical]` fallback forward) are shown with complete output in §7 and §10.

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

### 7.0 Methodology (seed once, drive the identical input many times)

The question "does the same reply resolve differently over time?" requires driving the **same unchanged input against the same persisted rows** — not regenerating users/aliases/contacts/IDs per run. `observe_dist.py` therefore separates **seed** from **drive**:

- **`seed <ab|ba> <on|off>`** truncates to a clean slate, seeds a **fixed** dataset (user_A `usera@mailbox.test`, user_B `userb@mailbox.test`, one alias each, and **two** `Contact` rows sharing `reply_email='dist-shared@sl.local'` with fixed `website_email`s `a-dist@nowhere.net` / `b-dist@nowhere.net`), sets both aliases' `disable_email_spoofing_check` per `on|off`, prints the physical `ctid` order and `EXPLAIN` of the exact lookup, then **stops**. `ab` vs `ba` controls only the contacts' **insertion order**.
- **`drive <N> <mid_prefix>`** does **not** reseed. It asserts exactly two contacts share the `reply_email` (else exits `3`), then drives the **canonical** `email_handler.handle_reply()` `N=20` times with an **identical** envelope — fixed `mail_from='usera@mailbox.test'` (user_A's mailbox) and fixed `rcpt_to='dist-shared@sl.local'` — where **only the Message-ID varies** (`<mid_prefix>-i@sl.local`). Every event is correlated to its own `EmailLog` by **exact Message-ID** (`EmailLog.get_by(message_id=mid)`), asserting `contact_id`/`alias_id`/`user_id`/`mailbox_id`/`is_reply` and the return `code`. All `N` events are aggregated into two distribution counters (by return code, and by resolved contact/alias/user or non-forward reason); for readability the harness prints the first three events and the last event as a **sample**, and the counters (which sum to `N=20`) are the authoritative full distribution. Argument bounds are validated (`order ∈ {ab,ba}`, `spoof ∈ {on,off}`, `N ∈ 1..1000`, `mid_prefix` matches `[A-Za-z0-9_.-]+`), with non-zero exits on invalid input.

**Every run performed against the clean-slate database is disclosed below** — four seeds and eight drive runs (each of the four conditions driven twice) — none omitted. The four conditions are the cross product of insertion order (`ab`/`ba`) and spoof control (`off` = **default**; `on` = **`[non-canonical]` fallback**).

The runner used for every invocation:

```
# /tmp/run_dist.sh <args>  (host-side wrapper; sources the canonical env, then runs the harness)
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_dist.py '"$1"' 2>&1'
```

### 7.1 Condition `ab` / `off` — default control, sender owns the first-inserted contact (correct user)

**Seed** (`/tmp/run_dist.sh "seed ab off"`):

```
2026-07-13 18:34:57,992 - SL - INFO - 6084 - "/app/init_app.py:44" - add_sl_domains() -  - Add d1.test to SL domain
2026-07-13 18:34:57,996 - SL - INFO - 6084 - "/app/init_app.py:44" - add_sl_domains() -  - Add d2.test to SL domain
2026-07-13 18:34:57,997 - SL - INFO - 6084 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
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

First-inserted `Contact id=1` (`user_A`) is at `ctid=(0,1)`; second at `ctid=(0,2)`. **Drive run 1** (`/tmp/run_dist.sh "drive 20 abD1"`) and **drive run 2** (`/tmp/run_dist.sh "drive 20 abD2"`), same persisted rows:

```
########## DRIVE run1 (N=20) — dataset ab/off ##########
=== DRIVE mode N=20 mid_prefix='abD1' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
driving handle_reply() 20x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local'; ONLY Message-ID varies
  event mid=<abD1-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abD1-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abD1-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abD1-19@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
--- distribution by return code ---
  code='250 Message accepted for delivery': 20
--- distribution by resolved (EmailLog) / non-forward ---
  contact_id=1,alias_id=3,user_id=1: 20
DRIVE DONE
```

```
########## DRIVE run2 (N=20, SAME persisted rows) — dataset ab/off ##########
=== DRIVE mode N=20 mid_prefix='abD2' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
driving handle_reply() 20x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local'; ONLY Message-ID varies
  event mid=<abD2-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abD2-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abD2-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abD2-19@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
--- distribution by return code ---
  code='250 Message accepted for delivery': 20
--- distribution by resolved (EmailLog) / non-forward ---
  contact_id=1,alias_id=3,user_id=1: 20
DRIVE DONE
```

**Result:** both runs **20/20** `code='250 Message accepted for delivery'`, all resolved `contact_id=1,alias_id=3,user_id=1` — the correct owner (user_A). Stable across the two runs.

### 7.2 Condition `ba` / `off` — default control, `.first()` resolves user_B → **E214 access-control rejection (no forward)**

**Seed** (`/tmp/run_dist.sh "seed ba off"`) — insertion order reversed, so `Contact id=1` is now `user_B`'s (at `ctid=(0,1)`):

```
########## SEED ba off ##########
=== SEED mode order=ba spoof=off (disable_email_spoofing_check=False) ===
shared reply_email='dist-shared@sl.local'  FIXED_MAIL_FROM='usera@mailbox.test'
user_a.id=1 alias_a.id=3  user_b.id=2 alias_b.id=4
insertion #1 -> Contact id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
insertion #2 -> Contact id=2 alias_id=3 user_id=1 website='a-dist@nowhere.net'
  ctid=(0,1) id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
  ctid=(0,2) id=2 alias_id=3 user_id=1 website='a-dist@nowhere.net'
--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---
  Limit  (cost=0.14..4.16 rows=1 width=4)
    ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..4.16 rows=1 width=4)
--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---
  Limit  (cost=0.00..11.00 rows=1 width=4)
    ->  Seq Scan on contact  (cost=0.00..11.00 rows=1 width=4)
SEED DONE (persisted; run 'drive' next)
```

**Drive run 1** (`drive 20 baD1`) and **drive run 2** (`drive 20 baD2`), same persisted rows, **identical** `mail_from='usera@mailbox.test'`:

```
########## DRIVE run1 (N=20) — dataset ba/off ##########
=== DRIVE mode N=20 mid_prefix='baD1' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
driving handle_reply() 20x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local'; ONLY Message-ID varies
  event mid=<baD1-0@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (resolved contact unauthorized for mail_from -> E214, no forward)
  event mid=<baD1-1@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (resolved contact unauthorized for mail_from -> E214, no forward)
  event mid=<baD1-2@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (resolved contact unauthorized for mail_from -> E214, no forward)
  event mid=<baD1-19@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (resolved contact unauthorized for mail_from -> E214, no forward)
--- distribution by return code ---
  code='250 SL E214 Unauthorized for using reverse alias': 20
--- distribution by resolved (EmailLog) / non-forward ---
  NO_EmailLog(code=250 SL E214 Unauthorized for using reverse alias): 20
DRIVE DONE
```

```
########## DRIVE run2 (N=20, SAME persisted rows) — dataset ba/off ##########
=== DRIVE mode N=20 mid_prefix='baD2' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
driving handle_reply() 20x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local'; ONLY Message-ID varies
  event mid=<baD2-0@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (resolved contact unauthorized for mail_from -> E214, no forward)
  event mid=<baD2-1@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (resolved contact unauthorized for mail_from -> E214, no forward)
  event mid=<baD2-2@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (resolved contact unauthorized for mail_from -> E214, no forward)
  event mid=<baD2-19@sl.local> delivered=False code='250 SL E214 Unauthorized for using reverse alias' EmailLog=None (resolved contact unauthorized for mail_from -> E214, no forward)
--- distribution by return code ---
  code='250 SL E214 Unauthorized for using reverse alias': 20
--- distribution by resolved (EmailLog) / non-forward ---
  NO_EmailLog(code=250 SL E214 Unauthorized for using reverse alias): 20
DRIVE DONE
```

**Result:** both runs **20/20** `code='250 SL E214 Unauthorized for using reverse alias'`, `EmailLog=None`. The identical input now resolves `user_B`'s contact (via `.first()` at `ctid=(0,1)`), user_A's mailbox is **not** authorized for user_B's alias, so under the **default** spoof control the reply is **rejected with E214 before any `EmailLog`/forward**. This is the crucial correction to the earlier draft: under default settings, wrong-user *resolution* manifests as an **access-control boundary**, not a wrong-user delivery.

### 7.3 Conditions `ab`/`on` and `ba`/`on` — `[non-canonical]` fallback (spoof check disabled) → the actual wrong-user forward

With `disable_email_spoofing_check = True` on both aliases (a **non-default** per-alias flag; labeled `[non-canonical]` fallback), the unauthorized-mailbox branch is skipped and the reply proceeds under the resolved contact's user.

**`ab`/`on` seed** and **drives** (`seed ab on`, `drive 20 abonD1`, `drive 20 abonD2`):

```
########## SEED ab on ([non-canonical] spoof disabled) ##########
=== SEED mode order=ab spoof=on (disable_email_spoofing_check=True) ===
shared reply_email='dist-shared@sl.local'  FIXED_MAIL_FROM='usera@mailbox.test'
user_a.id=1 alias_a.id=3  user_b.id=2 alias_b.id=4
insertion #1 -> Contact id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
insertion #2 -> Contact id=2 alias_id=4 user_id=2 website='b-dist@nowhere.net'
  ctid=(0,1) id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
  ctid=(0,2) id=2 alias_id=4 user_id=2 website='b-dist@nowhere.net'
--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---
  Limit  (cost=0.14..4.16 rows=1 width=4)
    ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..4.16 rows=1 width=4)
--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---
  Limit  (cost=0.00..11.00 rows=1 width=4)
    ->  Seq Scan on contact  (cost=0.00..11.00 rows=1 width=4)
SEED DONE (persisted; run 'drive' next)
```

```
########## DRIVE run1 (N=20) — dataset ab/on ##########
=== DRIVE mode N=20 mid_prefix='abonD1' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
driving handle_reply() 20x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local'; ONLY Message-ID varies
  event mid=<abonD1-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonD1-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonD1-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonD1-19@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
--- distribution by return code ---
  code='250 Message accepted for delivery': 20
--- distribution by resolved (EmailLog) / non-forward ---
  contact_id=1,alias_id=3,user_id=1: 20
DRIVE DONE
```

```
########## DRIVE run2 (N=20, SAME persisted rows) — dataset ab/on ##########
=== DRIVE mode N=20 mid_prefix='abonD2' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=3 user_id=1 website='a-dist@nowhere.net'
driving handle_reply() 20x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local'; ONLY Message-ID varies
  event mid=<abonD2-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonD2-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonD2-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
  event mid=<abonD2-19@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=3 user_id=1 mailbox_id=1 is_reply=True
--- distribution by return code ---
  code='250 Message accepted for delivery': 20
--- distribution by resolved (EmailLog) / non-forward ---
  contact_id=1,alias_id=3,user_id=1: 20
DRIVE DONE
```

**`ba`/`on` seed** and **drives** (`seed ba on`, `drive 20 baonD1`, `drive 20 baonD2`) — the wrong-user forward:

```
########## SEED ba on ([non-canonical] spoof disabled) ##########
=== SEED mode order=ba spoof=on (disable_email_spoofing_check=True) ===
shared reply_email='dist-shared@sl.local'  FIXED_MAIL_FROM='usera@mailbox.test'
user_a.id=1 alias_a.id=3  user_b.id=2 alias_b.id=4
insertion #1 -> Contact id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
insertion #2 -> Contact id=2 alias_id=3 user_id=1 website='a-dist@nowhere.net'
  ctid=(0,1) id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
  ctid=(0,2) id=2 alias_id=3 user_id=1 website='a-dist@nowhere.net'
--- EXPLAIN (default plan) of the lookup: filter_by(reply_email=...).first() ---
  Limit  (cost=0.14..4.16 rows=1 width=4)
    ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..4.16 rows=1 width=4)
--- EXPLAIN with index/bitmap scans DISABLED (forced Seq Scan), same query ---
  Limit  (cost=0.00..11.00 rows=1 width=4)
    ->  Seq Scan on contact  (cost=0.00..11.00 rows=1 width=4)
SEED DONE (persisted; run 'drive' next)
```

```
########## DRIVE run1 (N=20) — dataset ba/on ##########
=== DRIVE mode N=20 mid_prefix='baonD1' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
driving handle_reply() 20x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local'; ONLY Message-ID varies
  event mid=<baonD1-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonD1-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonD1-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonD1-19@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
--- distribution by return code ---
  code='250 Message accepted for delivery': 20
--- distribution by resolved (EmailLog) / non-forward ---
  contact_id=1,alias_id=4,user_id=2: 20
DRIVE DONE
```

```
########## DRIVE run2 (N=20, SAME persisted rows) — dataset ba/on ##########
=== DRIVE mode N=20 mid_prefix='baonD2' (dataset NOT reseeded) ===
[non-canonical] direct Contact.get_by(reply_email='dist-shared@sl.local') -> id=1 alias_id=4 user_id=2 website='b-dist@nowhere.net'
driving handle_reply() 20x with IDENTICAL mail_from='usera@mailbox.test', rcpt_to='dist-shared@sl.local'; ONLY Message-ID varies
  event mid=<baonD2-0@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonD2-1@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonD2-2@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
  event mid=<baonD2-19@sl.local> delivered=True code='250 Message accepted for delivery' EmailLog(mid match=True) contact_id=1 alias_id=4 user_id=2 mailbox_id=2 is_reply=True
--- distribution by return code ---
  code='250 Message accepted for delivery': 20
--- distribution by resolved (EmailLog) / non-forward ---
  contact_id=1,alias_id=4,user_id=2: 20
DRIVE DONE
```

**Result:** `ab`/`on` → both runs **20/20** E200 resolving `contact_id=1,alias_id=3,user_id=1` (user_A). `ba`/`on` → both runs **20/20** E200 resolving `contact_id=1,alias_id=4,user_id=2` — **user_B, the wrong user** (not the owner of the sending mailbox). The `EmailLog` records `user_id=2` for all 20 events per run. This is the condition under which a reply is actually *selected, logged, and enqueued for forwarding under the wrong user*. (As always, `NOT_SEND_EMAIL=true` means no external SMTP hand-off is confirmed — application acceptance and a stored send request only.)

### 7.4 PostgreSQL determinant: physical `ctid` and plan-dependence (bounded, honest interpretation)

Each seed printed the physical `ctid` order and the `EXPLAIN` of the exact lookup `SELECT contact.id FROM contact WHERE contact.reply_email='dist-shared@sl.local' LIMIT 1`:

- **Physical layout.** After a clean-slate truncate + fresh inserts, the two rows occupy the same page in insertion order: the first-inserted row is at `ctid=(0,1)`, the second at `ctid=(0,2)`. In condition `ab` the `(0,1)` row is user_A's; in `ba` it is user_B's.
- **Plans observed.** The default plan is `Index Scan using ix_contact_reply_email ... Limit rows=1`; with `enable_indexscan/bitmapscan/indexonlyscan` disabled, the same query plans a `Seq Scan on contact ... Limit rows=1`. (The absolute `cost=` figures printed above are PostgreSQL planner **estimates** that depend on table statistics / `ANALYZE` state and can vary run-to-run and across environments — e.g., on a later re-run of `seed ab off` the default `Index Scan` upper cost was `8.16` rather than the `4.16` captured here, while the forced `Seq Scan` stayed at `0.00..11.00` — whereas the plan **structure** (`Index Scan` vs forced `Seq Scan`, both `Limit rows=1`) and the physical `ctid` ordering were byte-for-byte stable. The interpretation below rests on the plan **structure** and physical order, **not** on the absolute cost estimate.)
- **What this does and does not prove.** The application specifies **no `ORDER BY`**; PostgreSQL is therefore free to return the matching rows in an unspecified order that, per its documentation, "*depend[s] on the scan and join plan types and the order on disk*" and "*must not be relied on*" (§9). For this **specific** two-row, single-page, append-only layout, both the observed `Index Scan` and the forced `Seq Scan` return the lower-`ctid` (first-inserted) row first, which is why every drive was **stable at 20/20** within a fixed layout. I therefore state only the **observed** fact — *no application ordering is specified, and the database-selected row here tracks the first-inserted/lower-`ctid` row under the active plan* — and treat the **general** claim, that a different plan, index state, page layout, `VACUUM`/update churn, or concurrency could return the *other* row, as explicitly **`[inferred]`** from the absence of an `ORDER BY` plus the official SQLAlchemy/PostgreSQL semantics (§9). The forced `Seq Scan` `EXPLAIN` demonstrates the plan can change; it does **not**, on this tiny layout, demonstrate a changed *result row*, and I do not claim it does.

### 7.5 Verdict

Across every disclosed run, the resolved contact — and therefore the alias, owning user, and mailbox derived from it — is determined by **which same-`reply_email` row `.first()` returns**, which under the active plan tracked the contacts' **insertion order**:

| Condition | Insertion order | `ctid=(0,1)` is | Spoof control | Drive result (run 1 / run 2, N=20 each) |
|-----------|-----------------|-----------------|---------------|------------------------------------------|
| `ab`/`off` | A-then-B | user_A (`id=1`) | **default** | 20/20 E200, `user_id=1` (correct) |
| `ba`/`off` | B-then-A | user_B (`id=1`) | **default** | 20/20 **E214**, `EmailLog=None` (rejected, **no forward**) |
| `ab`/`on` | A-then-B | user_A (`id=1`) | `[non-canonical]` | 20/20 E200, `user_id=1` (correct) |
| `ba`/`on` | B-then-A | user_B (`id=1`) | `[non-canonical]` | 20/20 E200, **`user_id=2` (wrong user forward)** |

- Within each fixed layout the outcome is **deterministic and stable** (20/20 across two runs) — it is *not* random per call.
- The resolved **user flips with insertion order** under the identical input, which is the ordering-dependence the unordered `.first()` allows.
- Under **default** settings the wrong-user resolution is caught as **E214** (no forward); the actual wrong-user **forward** is observed only under the **`[non-canonical]`** spoof-disabled fallback.

---
## 8. Edge / secondary conditions

All edge conditions are exercised through the **canonical** `handle_reply()` entry point (or, for `is_reverse_alias`, the labeled `[non-canonical]` predicate), with combined `stdout`+`stderr` captured (no suppression).

### 8.1 E501, E502 (no contact), E502 (inactive user), and the `is_reverse_alias` predicate

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_edge.py 2>&1'
```

**Output (complete, unedited; run 1 of 2 — run 2 byte-identical modulo per-run-varying fields only: the `RUN 1`/`RUN 2` banner, logger timestamps/PID, and the computed `delete_on` timestamp in the inactive-user case):**

```
########## observe_edge.py RUN 1 ##########
=== (1) E501: reply domain not EMAIL_DOMAIN and not an SLDomain ===
2026-07-13 18:40:21,149 - SL - WARNING - 6391 - "/app/email_handler.py:980" - handle_reply() -  - Reply email reply@notsl.example has wrong domain
rcpt_to='reply@notsl.example' -> delivered=False code='550 SL E501'  (==E501? True)
=== (2) E502: no Contact matches the reply_email ===
2026-07-13 18:40:21,156 - SL - WARNING - 6391 - "/app/email_handler.py:988" - handle_reply() -  - No contact with no-such-contact@sl.local as reverse alias
rcpt_to='no-such-contact@sl.local' -> delivered=False code='550 SL E502 Email not exist'  (==E502? True)
=== (3) E502: contact.user.is_active() is False (soft-deleted user) ===
inactive user delete_on=2026-07-14T18:40:21.125255+00:00  is_active()=False
2026-07-13 18:40:21,171 - SL - WARNING - 6391 - "/app/email_handler.py:991" - handle_reply() -  - User <User 2 Test User edge-inactive@mailbox.test> has been soft deleted
rcpt_to='edge-inactive-reply@sl.local' -> delivered=False code='550 SL E502 Email not exist'  (==E502? True)
=== (4) is_reverse_alias() predicate [app/email_utils.py:1156-1158] ===
is_reverse_alias('edge-active@sl.local')            = True
is_reverse_alias('not-a-reverse-alias@sl.local') = False
DONE
```

- **E501 — wrong reply domain.** `rcpt_to='reply@notsl.example'` does not end with `EMAIL_DOMAIN` and is not an `SLDomain`, so `handle_reply()` logs `Reply email reply@notsl.example has wrong domain` (`email_handler.py:980`) and returns `(False, '550 SL E501')`.
- **E502 — no matching contact.** `rcpt_to='no-such-contact@sl.local'` passes the domain gate but resolves no contact, logging `No contact with no-such-contact@sl.local as reverse alias` (`email_handler.py:988`) and returning `'550 SL E502 Email not exist'`.
- **E502 — inactive (soft-deleted) user.** A contact whose owning user has a future `delete_on` (so `User.is_active()` is `False` — `app/models.py:766-769`) resolves the contact but then fails the active-user gate, logging `User <...> has been soft deleted` (`email_handler.py:991`) and returning `'550 SL E502 Email not exist'`. The printed `delete_on=2026-07-14T...  is_active()=False` confirms the precondition.
- **`is_reverse_alias()` predicate `[non-canonical]`.** `is_reverse_alias('edge-active@sl.local') = True` (a contact exists), `is_reverse_alias('not-a-reverse-alias@sl.local') = False` (no contact and the address does not match the `reply+`/`ra+` suffix clause). This is the two-clause predicate from §6.2.

### 8.2 Normalization semantics — inbound spelling aliasing vs. exact-equality lookup (corrected)

This experiment establishes the **precise** normalization semantics and corrects the earlier conflation: `normalize_reply_email()` transforms only the **inbound** `rcpt_to`; the stored `Contact.reply_email` is compared by **exact equality**. Multiple inbound spellings therefore collapse onto **one** stored key, but a **multi-row** match still requires **more than one stored row** holding that exact key.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_norm.py 2>&1'
```

**Output (complete, unedited; run 1 of 2 — run 2 is byte-identical except for the per-run-varying fields: logger timestamps, PID, the randomly-generated alias local-part, and the `sl_message_id` tokens):**

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

### 8.3 Second lookup site — `replace_header_when_reply()` (`email_handler.py:364`)

During the reply, `handle_reply()` calls `replace_header_when_reply()` for the `TO` header (`email_handler.py:1179`) and the `CC` header (`email_handler.py:1181`); that function performs a **second** `Contact.get_by(reply_email=...)` (`email_handler.py:364`) to restore each reverse-alias in the header back to the original contact address (or raises `NonReverseAliasInReplyPhase` if none matches). This is a second site subject to the same non-unique `.first()` semantics.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python /tmp/observe_second.py 2>&1'
```

**Output (complete, unedited; run 1 of 2 — run 2 byte-identical modulo per-run-varying fields only: the `RUN 1`/`RUN 2` banner and logger timestamps/PID):**

```
########## observe_second.py RUN 1 ##########
=== second lookup site: replace_header_when_reply() [email_handler.py:364] via TO & CC ===
primary rcpt_to='second-r1@sl.local' (resolved at :986); TO header='second-r1@sl.local' -> contact c1.id=1 website='w1@nowhere.net'
CC header='second-r2@sl.local' -> contact c2.id=2 website='w2@nowhere.net'
2026-07-13 18:43:38,093 - SL - DEBUG - 6532 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: second-r1@sl.local, new: W1 <w1@nowhere.net>
2026-07-13 18:43:38,095 - SL - DEBUG - 6532 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace Cc header, old: second-r2@sl.local, new: W2 <w2@nowhere.net>
handle_reply -> delivered=True code='250 Message accepted for delivery' (==E200? True)
EmailLog id=1 contact_id=1 alias_id=2 user_id=1 is_reply=True
rewritten msg[To] = 'W1 <w1@nowhere.net>'
rewritten msg[Cc] = 'W2 <w2@nowhere.net>'
DONE
```

- The canonical `handle_reply()` call carries a `TO` header `second-r1@sl.local` (resolving contact `c1.id=1`, `w1@nowhere.net`) and a `CC` header `second-r2@sl.local` (resolving contact `c2.id=2`, `w2@nowhere.net`). The second-site lookup rewrites both: `Replace To header, old: second-r1@sl.local, new: W1 <w1@nowhere.net>` and `Replace Cc header, old: second-r2@sl.local, new: W2 <w2@nowhere.net>` (`email_handler.py:380`).
- Result: `delivered=True code='250 Message accepted for delivery'` (E200), `EmailLog id=1 contact_id=1 alias_id=2 user_id=1 is_reply=True`, and the rewritten `msg[To]='W1 <w1@nowhere.net>'`, `msg[Cc]='W2 <w2@nowhere.net>'`. Two **distinct** contacts are resolved through the second site in one reply, confirming the site is live on the canonical path.

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

**Output (complete, unedited; run 1 of 2 — run 2 byte-identical modulo per-run-varying fields only: logger timestamps/PID and the randomly-generated alias local-parts):**

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

**Causal conclusion.** The wrong-user *resolution* is caused by the unordered `.first()` over a non-unique `reply_email` (§4.1, §6.1–§6.3). Whether that mis-resolution becomes a wrong-user **forward** depends on the alias's spoof control: **default** settings convert it into an E214 rejection plus an alert to the resolved (wrong) user (CASE 1); the **`[non-canonical]`** spoof-disabled fallback lets it become an actual forward logged against the wrong user (CASE 2). In neither case is external SMTP delivery confirmed under this harness configuration.

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

All ten scripts are reproduced verbatim (they live outside the repository and were deleted afterward — absence proof in §13). Each of the nine Python harnesses shares the fail-fast bootstrap shown in §2.7; the tenth, `contact_schema.sql`, is a plain `psql` script (no bootstrap). The exact invocation precedes each source.

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

**`observe_reply.py`** — canonical happy-path single call (§3–§5). Invocation:
```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_reply.py 2>&1'
```

```python
"""observe_reply.py — CANONICAL single-call baseline for the SimpleLogin reply path.

Seeds two Contacts (owned by different users) sharing ONE reply_email, in
insertion order A-then-B, then drives the real inbound entry point
email_handler.handle_reply() once with mail_from = user_A's mailbox. First-inserted
contact belongs to user_A and mail_from matches user_A's mailbox => CORRECT-resolution
happy path. Captures the full per-event contract (derivation, resolved contact,
alias/user/mailbox, EmailLog correlated by EXACT Message-ID, stored outbound
SendRequest incl. its real external recipient). Runs inside the canonical container.
"""
import os
import sys

# (1) FAIL-FAST DISPOSABLE-DB ALLOWLIST — before importing app modules / any write.
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

# (2) Second fail-fast: the connected database must be the disposable 'test' DB.
with engine.connect() as _c:
    _dbname = _c.execute("select current_database()").scalar()
if _dbname != "test":
    sys.stderr.write(f"REFUSING TO RUN: current_database()={_dbname!r} != 'test'\n")
    sys.exit(3)

# (3) pg_trgm already exists (alembic); ensure idempotently WITHOUT DROP.
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


mail_sender.store_emails_instead_of_sending(True)

with app.app_context():
    truncate_clean_slate()      # clean slate FIRST (before any Session query)
    add_sl_domains()            # re-seed SLDomain rows (domain gate needs sl.local)
    add_proton_partner()

    shared_reply = f"dup-reply-core@{EMAIL_DOMAIN}"
    user_a = create_new_user(email="usera@mailbox.test")
    user_b = create_new_user(email="userb@mailbox.test")
    Session.commit()
    alias_a = Alias.create_new_random(user_a)
    alias_b = Alias.create_new_random(user_b)
    Session.commit()

    # INSERTION ORDER A-then-B: user_A's contact is inserted first.
    contact_a = Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id,
                               website_email="a@nowhere.net", name="A", reply_email=shared_reply)
    contact_b = Contact.create(user_id=alias_b.user_id, alias_id=alias_b.id,
                               website_email="b@nowhere.net", name="B", reply_email=shared_reply)
    Session.commit()

    print("=== SEED (insertion order A-then-B) ===")
    print(f"EMAIL_DOMAIN={EMAIL_DOMAIN!r}  NOT_SEND_EMAIL={NOT_SEND_EMAIL!r}")
    print(f"shared_reply={shared_reply!r}")
    print(f"user_a.id={user_a.id} email={user_a.email!r} alias_a.id={alias_a.id} alias_a.email={alias_a.email!r}")
    print(f"user_b.id={user_b.id} email={user_b.email!r} alias_b.id={alias_b.id} alias_b.email={alias_b.email!r}")
    print(f"contact_a.id={contact_a.id} (alias_a,user_a)  contact_b.id={contact_b.id} (alias_b,user_b)")

    print("=== PROVE DUPLICATE PERSISTED (no UNIQUE constraint blocked it) ===")
    rows = Contact.filter_by(reply_email=shared_reply).all()
    print(f"rows sharing reply_email={shared_reply!r}: count={len(rows)}")
    for r in rows:
        print(f"  Contact id={r.id} alias_id={r.alias_id} user_id={r.user_id} website_email={r.website_email!r}")

    print("=== reply-address derivation values (email_handler.py:972,977,984) ===")
    print(f"rcpt_to (raw)              = {shared_reply!r}")
    print(f"endswith(EMAIL_DOMAIN)     = {shared_reply.endswith(EMAIL_DOMAIN)}")
    print(f"normalize_reply_email(...) = {normalize_reply_email(shared_reply)!r}")

    print("=== [non-canonical] routing predicate + direct lookup (BYPASS the entry point) ===")
    print(f"[non-canonical] is_reverse_alias({shared_reply!r}) = {is_reverse_alias(shared_reply)}")
    d = Contact.get_by(reply_email=shared_reply)
    print(f"[non-canonical] Contact.get_by(reply_email=...) -> id={d.id} alias_id={d.alias_id} user_id={d.user_id}")

    mid = "<obs-core-0@sl.local>"
    msg = EmailMessage()
    msg["From"] = "a@nowhere.net"
    msg["To"] = alias_a.email
    msg["Message-ID"] = mid
    msg["Subject"] = "reply subject"
    msg.set_content("hello body")
    envelope = Envelope()
    envelope.mail_from = user_a.email
    envelope.rcpt_tos = [shared_reply]

    mail_sender.purge_stored_emails()
    alert_before = Session.query(SentAlert).count()

    print("=== CANONICAL CALL: email_handler.handle_reply(envelope, msg, shared_reply) ===")
    print(f"envelope.mail_from={envelope.mail_from!r} envelope.rcpt_tos={envelope.rcpt_tos!r} Message-ID={mid!r}")
    delivered, code = email_handler.handle_reply(envelope, msg, shared_reply)
    print(f"RESULT delivered={delivered!r} code={code!r}")
    print(f"code==status.E200? {code == status.E200}  ==E214? {code == status.E214}  ==E502? {code == status.E502}")

    el = EmailLog.get_by(message_id=mid)
    if el:
        print(f"PERSISTED EmailLog (message_id={mid!r}): id={el.id} contact_id={el.contact_id} "
              f"alias_id={el.alias_id} user_id={el.user_id} mailbox_id={el.mailbox_id} is_reply={el.is_reply}")
        print(f"  -> forwarding user_id={el.user_id} (user_a.id={user_a.id}, user_b.id={user_b.id})")
    else:
        print(f"No EmailLog for message_id={mid!r} (non-success path).")

    stored = mail_sender.get_stored_emails()
    print(f"stored SendRequest count = {len(stored)}")
    for sr in stored:
        print(f"  SendRequest envelope_from={sr.envelope_from!r} envelope_to={sr.envelope_to!r} "
              f"msg[From]={sr.msg[headers.FROM]!r} msg[To]={sr.msg[headers.TO]!r}")
    print("NOTE: NOT_SEND_EMAIL=%r => mail_sender.send() logs and returns True WITHOUT calling _send_to_smtp;"
          " no external SMTP delivery is confirmed (app/mail_sender.py:130-136)." % NOT_SEND_EMAIL)

    alert_after = Session.query(SentAlert).count()
    print(f"NEW SentAlert rows during call: {alert_after - alert_before}")
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

**`observe_dist.py`** — same-unchanged-input distribution, seed-once/drive-many (§7). Invocations: `seed <ab|ba> <on|off>` and `drive <N> <mid_prefix>` via `/tmp/run_dist.sh` (§7.0).

```python
"""observe_dist.py — Cross-event distribution under the SAME UNCHANGED INPUT.

Two modes, so the dataset is SEEDED ONCE and then the IDENTICAL input is DRIVEN
many times against those same persisted rows (only Message-ID varies):

  seed <ab|ba> <on|off>
      Truncate (allowlist-guarded), seed a FIXED dataset: user_A/user_B, one alias
      each, and TWO Contact rows sharing the SAME reply_email (fixed website_emails).
      <ab|ba> controls contact INSERTION ORDER. <on|off> sets both aliases'
      disable_email_spoofing_check. Prints ctid physical order + EXPLAIN (default and
      forced Seq Scan) for the exact lookup query. Persists and STOPS.

  drive <N> <mid_prefix>
      Does NOT reseed. Asserts exactly two contacts share the reply_email (else exit 3).
      Drives handle_reply() N times with an IDENTICAL envelope (fixed mail_from and
      rcpt_to); ONLY Message-ID varies (= "<mid_prefix>-i@sl.local"). Correlates each
      event by exact Message-ID via EmailLog and prints a full per-event contract plus
      distribution counters.

Arg validation (#25): invalid args exit non-zero (2 = bad args, 3 = data precondition).
"""
import os
import re
import sys

_ALLOWED_DSNS = {
    "postgresql://test:test@localhost:5432/test",
    "postgresql://test:test@localhost:15432/test",
    "postgresql://test:test@127.0.0.1:5432/test",
}
if os.environ.get("DB_URI", "") not in _ALLOWED_DSNS:
    sys.stderr.write(f"REFUSING TO RUN: DB_URI={os.environ.get('DB_URI','')!r} not in disposable allowlist\n")
    sys.exit(3)

# ---- bounded/validated CLI parsing BEFORE any heavy import (#25) ----
SHARED_REPLY = "dist-shared@sl.local"
FIXED_MAIL_FROM = "usera@mailbox.test"   # user_A's mailbox (fixed across every event)
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
from app.models import Alias, Contact, EmailLog
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

        direct = Contact.get_by(reply_email=SHARED_REPLY)
        print(f"=== DRIVE mode N={N} mid_prefix={MID_PREFIX!r} (dataset NOT reseeded) ===")
        print(f"[non-canonical] direct Contact.get_by(reply_email={SHARED_REPLY!r}) -> "
              f"id={direct.id} alias_id={direct.alias_id} user_id={direct.user_id} website={direct.website_email!r}")
        print(f"driving handle_reply() {N}x with IDENTICAL mail_from={FIXED_MAIL_FROM!r}, "
              f"rcpt_to={SHARED_REPLY!r}; ONLY Message-ID varies")

        by_code = {}
        by_contact = {}
        for i in range(N):
            mid = f"<{MID_PREFIX}-{i}@sl.local>"
            msg = EmailMessage()
            msg["From"] = "someone@nowhere.net"
            msg["To"] = SHARED_REPLY
            msg["Message-ID"] = mid
            msg["Subject"] = f"dist {i}"
            msg["Date"] = "Mon, 13 Jul 2026 00:00:00 +0000"
            msg.set_content("body")
            env = Envelope()
            env.mail_from = FIXED_MAIL_FROM
            env.rcpt_tos = [SHARED_REPLY]
            delivered, code = email_handler.handle_reply(env, msg, SHARED_REPLY)
            Session.commit()
            by_code[code] = by_code.get(code, 0) + 1
            el = EmailLog.get_by(message_id=mid)
            if el:
                key = f"contact_id={el.contact_id},alias_id={el.alias_id},user_id={el.user_id}"
                by_contact[key] = by_contact.get(key, 0) + 1
                if i < 3 or i == N - 1:
                    print(f"  event mid={mid} delivered={delivered} code={code!r} "
                          f"EmailLog(mid match={el.message_id==mid}) contact_id={el.contact_id} "
                          f"alias_id={el.alias_id} user_id={el.user_id} mailbox_id={el.mailbox_id} is_reply={el.is_reply}")
            else:
                key = f"NO_EmailLog(code={code})"
                by_contact[key] = by_contact.get(key, 0) + 1
                if i < 3 or i == N - 1:
                    print(f"  event mid={mid} delivered={delivered} code={code!r} EmailLog=None "
                          f"(resolved contact unauthorized for mail_from -> E214, no forward)")

        print("--- distribution by return code ---")
        for k in sorted(by_code):
            print(f"  code={k!r}: {by_code[k]}")
        print("--- distribution by resolved (EmailLog) / non-forward ---")
        for k in sorted(by_contact):
            print(f"  {k}: {by_contact[k]}")
        print("DRIVE DONE")


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

**`contact_schema.sql`** — SQL catalog helper backing the §6.1(b) runtime proof that `reply_email` carries no `UNIQUE` constraint. Unlike the nine Python harnesses above it is a plain `psql` script (no app bootstrap and no writes): read-only catalog queries against `pg_indexes`, `pg_constraint`, and `pg_index`/`pg_class`/`pg_attribute`. Invocation:
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
| 7 | Same reply resolves to different contacts across events? | **OBSERVED** (flips with insertion order; deterministic/stable within a fixed layout, 20/20 ×2) | §7 |
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

**Net:** every part of the question is answered from runtime observation, with two items explicitly bounded as `[inferred]` (a different plan/layout returning the other row; a live concurrency race) and one explicitly **not** claimed (confirmed external SMTP delivery under `NOT_SEND_EMAIL=true`). These bounds are stated wherever the related claims appear, not only here.

---

## 13. Repository invariant & cleanup proof

**Cleanup — explicit per-file absence proof.** Every temporary harness lived in `/tmp` (host `/tmp/harness/`, container `/tmp/`), outside the repository. After capturing evidence they were removed; a bounded `test ! -e` per named script confirms absence (the command prints `ABSENT` for each and exits 0), alongside a `find` that returns nothing:

**Command:**

```
for f in _bootstrap observe_reply observe_handle observe_wronguser observe_dist \
         observe_edge observe_norm observe_avail observe_second; do
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
ABSENT(host): /tmp/harness/observe_second.py
ABSENT(container): /tmp/observe_second.py
--- find (host) ---
--- ls (container) ---
ls: cannot access '/tmp/_bootstrap.py': No such file or directory
ls: cannot access '/tmp/observe_*.py': No such file or directory
```

The tenth temporary script — the `psql` helper `contact_schema.sql` (source in §11.3) that produced the §6.1(b) catalog proof — was likewise removed after use; its absence is confirmed the same way (`.sql`, not `.py`, so it is checked separately):

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
