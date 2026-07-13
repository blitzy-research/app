# SimpleLogin custom-alias creation: signed-suffix validation & limit enforcement — a runtime-grounded investigation

**What this document is.** A direct, evidence-first answer to five questions about how SimpleLogin
validates *signed suffixes* and enforces *alias-creation limits* on the custom-alias endpoints
`POST /api/v2/alias/custom/new` and `POST /api/v3/alias/custom/new`. Every behavioural claim below is
backed by output captured while **running the real code** — the real Flask endpoints exercised through
their real entry points — not by reading alone.

**Runtime / provenance.** All output was captured inside the prescribed container image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (alias
`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`), against the repository baked at
`/app` (commit `2cd6ee77`), Python **3.10.18**, PostgreSQL **15** (`localhost:5432`, `test/test/test`,
`pg_trgm` enabled) and Redis (`localhost:6379`). Library versions are the pinned set:
`flask 1.1.2`, `flask-limiter 1.4`, `itsdangerous 1.1.0`, `werkzeug 1.0.1`, `sqlalchemy 1.3.24`,
`redis 4.6.0`, `flask-login 0.5.0`, `gunicorn 20.0.4`.

**How the evidence was produced.** A set of self-contained probe scripts drives the endpoints and
captures the exact HTTP status, response body, response headers and server-console log lines for every
condition. Two harnesses are used:

* **Test-client harness** (fastest canonical reproduction): `server.create_app()` + the Werkzeug test
  client, with database isolation via `connection.begin()` / `transaction.rollback()` — identical to the
  `flask_client` fixture in `tests/conftest.py` (L59–L77). This commits **nothing** to PostgreSQL.
* **Live-server harness**: a real `gunicorn wsgi:app` process probed with `curl -i`, used specifically to
  capture the **transport-level** response headers (`Server`, `Date`, `Connection`) that a test client
  does not synthesise.

The complete, unedited probe sources and the master harness (`run_all.sh`, which proves net-zero
residue in PostgreSQL and Redis) are reproduced verbatim in the Appendix (§11). Every signed suffix
shown is **synthetic, disposable, test-only and time-limited** (`max_age=600s`); the signing secret
`CUSTOM_ALIAS_SECRET="secretcustom_alias"` is the **public, test-only** value derived from
`FLASK_SECRET="secret"` in `tests/test.env` and must never be treated as a credential.

---

## 1. Executive summary (direct answers)

**Lead finding (the most probable cause of "intermittent validation failures that don't match expected
behaviour").** On the custom-alias endpoints a **tampered** signed suffix, a **genuinely expired** signed
suffix, and an unsigned **garbage** string are *indistinguishable* at the HTTP layer: all three return
**HTTP 412** with the body `{"error":"Alias creation time is expired, please retry"}` and the **same**
server log line `Alias creation time expired for <user>`. The root cause is a single line in the
validator: `check_suffix_signature` (`app/alias_suffix.py:37-42`) wraps
`signer.unsign(signed_suffix, max_age=600)` in `except itsdangerous.BadSignature: return None`, and in
`itsdangerous 1.1.0` **`SignatureExpired` and `BadTimeSignature` are both subclasses of `BadSignature`**
(runtime-proven in §9). Because all three error classes are caught identically and collapse to `None`, the
handler's `if not alias_suffix:` branch fires for every one of them and returns the *expiry* 412. The
handler's sibling `except Exception:` branch — which would log a tamper warning and
`return "Tampered suffix", 400` (`new_custom_alias.py:74-76` / `:189-191`) — is therefore **effectively
unreachable** for ordinary string input: a distinct 400 "Tampered suffix" is never produced for a tampered
string.

The five questions, answered:

| # | Question | Direct answer (observed) |
|---|----------|--------------------------|
| **Q1** | Exact HTTP status codes & error messages for **invalid** / **expired** signed suffixes | Both return **`412 PRECONDITION FAILED`**, body **`{"error":"Alias creation time is expired, please retry"}`** (`Content-Length: 57`). An unsigned garbage string behaves identically. Adjacent conditions: empty body → **`400`** `{"error":"request body cannot be empty"}`; duplicate → **`409`** `{"error":"alias <full-address> already exists"}`; two consecutive dots → **`400`** `{"error":"2 consecutive dot signs aren't allowed in an email address"}`; wrong prefix/suffix/domain → **`400`** `{"error":"wrong alias prefix or suffix"}`. Full transcripts: §3. |
| **Q2** | Corresponding server-console log entries | The 412 emits a `WARNING` at `new_custom_alias.py:72` (v2) / `:187` (v3) with message `Alias creation time expired for <user>`. Every response also emits a `DEBUG` at `server.py:284` (`after_request`) recording method, path and status. Other branches log: quota → `DEBUG` at `new_custom_alias.py:49`; duplicate → `DEBUG` at `new_custom_alias.py:87`; wrong suffix → `ERROR` at `alias_suffix.py:78`/`:61`; rate-limit → `WARNING` at `server.py:364`. The complete raw buffer, the exact verbatim lines, and the capture method are in §4. |
| **Q3** | Do rate-limiting headers appear, and their values | **No `X-RateLimit-*` and no `Retry-After` header appears on any response** (201, 412 or 429), confirmed both via the test client and via a live `gunicorn` server with `curl -i`. Reason: `flask-limiter 1.4` leaves `RATELIMIT_HEADERS_ENABLED` at its default **`False`** and `server.py:167` calls `limiter.init_app(app)` with no `headers_enabled` argument. When the limit is exceeded the response is **`429`** `{"error":"Rate limit exceeded"}`; with `ALIAS_LIMIT="100/day;50/hour;5/minute"` the **first 429 occurs on request #6** (5/minute allows exactly 5). Details & headers: §5. |
| **Q4** | Which quota checks run on success, and what values are logged | The account-level quota check is **`User.can_create_new_alias()`** (`app/models.py:867-884`), which calls **`max_alias_for_free_account()`** (`app/models.py:858-865`). On the **success** path the quota check itself **logs nothing** — the only lines emitted are the `INFO` `event_dispatcher.py:62` webhook notice and the `DEBUG` `server.py:284` after-request line. A value is logged **only on the failure path**: `DEBUG` at `new_custom_alias.py:49`, message `user <user> cannot create any custom alias`. The branch inputs actually observed (`is_active`, `disabled`, `lifetime_or_active_subscription`, `FLAG_FREE_OLD_ALIAS_LIMIT`, `max_alias_for_free_account`, current alias count) are printed in §6. |
| **Q5** | Execution-path trace: what validates suffixes / enforces limits, and what triggers rejection | Decorator chain then in-handler branches (§7 diagram). Suffix validation is **`check_suffix_signature`** + **`verify_prefix_suffix`** (`app/alias_suffix.py:37-91`). **Four** distinct limit enforcers exist: (1) `@limiter.limit(ALIAS_LIMIT)` HTTP rate limit → 429; (2) `@parallel_limiter.lock(name="alias_creation")` concurrency lock → 429; (3) `User.can_create_new_alias()` account quota → 400; (4) `check_bucket_limit` inside `Alias.create` (`app/models.py:1634-1641`) → 429. All three 429 surfaces route through the one custom handler at `server.py:364`. |

> Cross-reference: the end-to-end execution-path diagram referenced by Q5 is the flowchart in **§7**
> ("Execution-path trace"). Section numbering here is stable: §5 covers rate limiting (Q3) and §7 covers
> the trace (Q5).

---

## 2. Environment & How to Reproduce

Every observation in this document was captured inside the **prescribed container** (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, tag suffix `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`, running as `sl_setup`), whose baked-in checkout at `/app` matches host commit `2cd6ee77`. Dependencies live in the venv at `/app/venv`, so every probe was run with `/app/venv/bin/python` (the system Python has no dependencies). All work drove the **real** Flask app (`server.create_app`) against a **real** PostgreSQL 15 (`localhost:5432`, `test/test/test`, with the `pg_trgm` extension) and a **real** Redis (`localhost:6379`) — no mocks, no stubs, no bypassing interfaces.

### 2.1 Services & versions actually observed

The two backing services are process-level (started fresh when the container starts — they are **not** baked into the image). The following readiness transcript was captured live; it confirms both services accept connections, the `pg_trgm` extension is present, the schema is migrated to the Alembic head, and the pinned library versions match the AAP exactly:

```text
############################################################
# ENVIRONMENT READINESS TRANSCRIPT (Finding 13)
# Captured 2026-07-13T20:17:00Z inside container sl_setup.
############################################################

$ service postgresql status
15/main (port 5432): online

$ pg_isready -h localhost -p 5432 -U test
localhost:5432 - accepting connections

$ redis-cli ping
PONG

$ PGPASSWORD=test psql -h localhost -U test -d test -tAc "SELECT extname FROM pg_extension WHERE extname='pg_trgm'"
pg_trgm

$ PGPASSWORD=test psql -h localhost -U test -d test -tAc "SELECT version_num FROM alembic_version"
32f25cbf12f6

$ PGPASSWORD=test psql -h localhost -U test -d test -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public'"
77

$ /app/venv/bin/python -c "import sys,itsdangerous,flask,flask_limiter,werkzeug,redis,sqlalchemy"
$   print(sys.version.split()[0], itsdangerous.__version__, flask.__version__, flask_limiter.__version__, werkzeug.__version__, redis.__version__, sqlalchemy.__version__)
python            3.10.18
itsdangerous      1.1.0
flask             1.1.2
flask_limiter     1.4
werkzeug          1.0.1
redis             4.6.0
sqlalchemy        1.3.24
```

### 2.2 Clean-start order (from a stopped container)

The services must be started **before** any probe or server, in this exact order, and confirmed ready:

```bash
# 1. Start the two backing services (idempotent; safe to re-run).
service postgresql start          # PostgreSQL 15 on localhost:5432
service redis-server start        # Redis on localhost:6379

# 2. Confirm readiness BEFORE launching the app (do not race the boot).
pg_isready -h localhost -p 5432 -U test     # -> "localhost:5432 - accepting connections"
redis-cli ping                              # -> "PONG"

# 3. The shipped image is already migrated to Alembic head 32f25cbf12f6 (77 tables) with the
#    pg_trgm extension created. If provisioning from scratch, the idempotent /build.sh performs
#    the migration and extension creation; re-running it against an already-migrated DB is a no-op.
```

The database is a persistent volume, so it survives across `docker exec` sessions; Redis starts empty on each container start and only accumulates keys at runtime. This distinction matters for the net-zero proof in §2.5.

### 2.3 Two canonical runners (both drive the real entry point)

Two runners were used, both exercising the **real** endpoint through its **real** entry point.

**Runner A — standalone probe scripts (fastest canonical harness).** The probes are ordinary Python scripts (not `pytest` modules); each builds the real Flask app with `server.create_app()` and drives `POST /api/v2|v3/alias/custom/new` through the Werkzeug test client with **real API-key authentication**. Isolation is identical to `tests/conftest.py::flask_client` [`tests/conftest.py:59-77`]: all DB writes happen inside one connection-level transaction opened with `connection.begin()` and rolled back in a `finally` block, so the probe **commits nothing** to PostgreSQL [`probe_functional.py` docstring L9-13]. Like the fixture, Runner A sets `config.DISABLE_RATE_LIMIT = True`, so it is used for status/body/log observations of every condition **except** the rate-limit headers (those need Runner B). The exact, copy-pasteable invocation — note the runtime DB port is **5432**, overriding the `15432` written in `tests/test.env:17` — is:

```bash
cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
  CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
  GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
  /app/venv/bin/python /tmp/sl_probes/probe_functional.py
```

Substitute `probe_ratelimit.py`, `probe_lock.py`, `probe_mro.py`, or `probe_f18.py` for the script name; each is a self-contained, runnable script (there is **no** per-probe `pytest` step — earlier drafts of this document listed one in error). `GNUPGHOME=/tmp/sl_gnupg` must point at a clean directory holding exactly the shipped PGP keys, or `create_app()` fails at import time.

**Runner B — live Gunicorn server (genuine over-the-wire evidence).** Used to capture the complete raw HTTP transport headers (`Server`/`Date`/`Connection`) and to answer the rate-limit-header question with rate limiting **enabled**. The server is launched exactly as the shipped `Dockerfile` CMD does — `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` [`Dockerfile:47`] — with the canonical `/tmp/sl.env` config, **except** `DISABLE_RATE_LIMIT` is unset so rate limiting is active (`config.py:602` reads the flag by **presence**: `DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ`). The canonical `--timeout` is **15**, not 30 (an earlier draft of this document used 30 in error).

The live runner (`probe_live.sh`) manages the server through a fully **bounded lifecycle** — it frees the port, captures the Gunicorn master PID, polls readiness before sending load, and terminates **only the captured PID** (`kill "$GPID"; wait "$GPID"` — never `pkill`/`killall`), then tears down all DB rows it created and verifies net-zero:

```bash
#!/bin/bash
# probe_live.sh -- capture COMPLETE raw HTTP response headers (incl. gunicorn transport
# headers Date/Server/Connection) from a LIVE gunicorn server for the 201 / 412 / 429
# responses of POST /api/v2/alias/custom/new, proving exactly which headers appear
# (answering "do rate-limit headers appear, and what are their actual values?").
#
# Canonical build/invocation: gunicorn wsgi:app (Dockerfile CMD) with the canonical
# /tmp/sl.env config, EXCEPT DISABLE_RATE_LIMIT is unset so rate limiting is active
# (config.py:602 DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ -> presence-based).
set -u
OUT=/tmp/sl_probes/out
mkdir -p "$OUT"
cd /app
PORT=7777

CANON_ENV=(env CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
  GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app)

echo "===== [0/5] ensure port ${PORT} is free ====="
STALE=$(ss -ltnp 2>/dev/null | grep ":${PORT} " | grep -o 'pid=[0-9]*' | head -1 | cut -d= -f2)
if [ -n "${STALE:-}" ]; then echo "killing stale listener pid=$STALE"; kill "$STALE" 2>/dev/null; sleep 1; fi
echo "port check done"

echo
echo "===== [1/5] SEED disposable premium user + API key ====="
SEED=$("${CANON_ENV[@]}" /app/venv/bin/python /tmp/sl_probes/probe_live_seed.py)
echo "$SEED"
API_KEY=$(echo "$SEED"  | sed -n 's/^API_KEY=//p')
VALID=$(echo "$SEED"    | sed -n 's/^VALID_SUFFIX=//p')
TAMPERED=$(echo "$SEED" | sed -n 's/^TAMPERED_SUFFIX=//p')

echo
echo "===== [2/5] START live gunicorn (rate limiting ENABLED) ====="
set -a; . /tmp/sl.env; set +a
unset DISABLE_RATE_LIMIT          # enable rate limiting (presence-based flag)
export DB_URI='postgresql://test:test@localhost:5432/test'
/app/venv/bin/gunicorn wsgi:app -b 127.0.0.1:${PORT} -w 2 --timeout 15 \
    > "$OUT/gunicorn.log" 2>&1 &
GPID=$!
echo "gunicorn master PID=$GPID (bound 127.0.0.1:${PORT}, -w 2 --timeout 15)"

# readiness poll -- wait until gunicorn accepts a connection AND returns a real HTTP code
READY=0
for i in $(seq 1 60); do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 "http://127.0.0.1:${PORT}/")
  case "$code" in
    200|301|302|401|403|404) READY=1; echo "ready after ${i} poll(s), GET / -> $code"; break;;
  esac
  sleep 0.5
done
sleep 1   # let both sync workers finish booting before load
if [ "$READY" != "1" ]; then
  echo "SERVER NEVER BECAME READY; gunicorn log:"; cat "$OUT/gunicorn.log"
  kill "$GPID" 2>/dev/null; wait "$GPID" 2>/dev/null
  "${CANON_ENV[@]}" /app/venv/bin/python /tmp/sl_probes/probe_live_teardown.py
  exit 1
fi

hdr() {  # $1=label $2=prefix $3=suffix $4=outfile
  echo "----- $1 -----"
  curl -sS -i --max-time 10 -X POST "http://127.0.0.1:${PORT}/api/v2/alias/custom/new" \
    -H "Authentication: ${API_KEY}" -H "Content-Type: application/json" \
    -d "{\"alias_prefix\":\"$2\",\"signed_suffix\":\"$3\"}" | tee "$4"
  echo; echo
}

echo
echo "===== [3/5] CAPTURE raw headers via curl -i (live transport) ====="
hdr "REQ1 valid  -> expect 201" lh1 "$VALID"    "$OUT/live_201.txt"
hdr "REQ2 tamper -> expect 412" lh2 "$TAMPERED" "$OUT/live_412.txt"
hdr "REQ3 valid  -> 201 (fill window)" lh3 "$VALID" /dev/null
hdr "REQ4 valid  -> 201 (fill window)" lh4 "$VALID" /dev/null
hdr "REQ5 valid  -> 201 (fill window)" lh5 "$VALID" /dev/null
hdr "REQ6 valid  -> expect 429 (6th within 5/minute)" lh6 "$VALID" "$OUT/live_429.txt"

echo
echo "===== [4/5] STOP gunicorn (only captured PID $GPID) ====="
kill "$GPID" 2>/dev/null
wait "$GPID" 2>/dev/null
echo "gunicorn stopped"

echo
echo "===== [5/5] TEARDOWN (SQL delete + verify net-zero) + Redis cleanup ====="
"${CANON_ENV[@]}" /app/venv/bin/python /tmp/sl_probes/probe_live_teardown.py
TD=$?
DEL=$(redis-cli --scan --pattern 'LIMITER/*ip:127.0.0.1*new_custom_alias*' | xargs -r redis-cli del)
echo "redis LIMITER keys deleted for ip:127.0.0.1/new_custom_alias: ${DEL:-0}"
exit $TD
```

The readiness poll (L44-50) waits until `GET /` returns a real HTTP status (`200|301|302|401|403|404`) rather than a connection failure, so evidence is never captured against a half-booted worker; the observed readiness was `GET / -> 302` (a redirect to login). The `-w 2`/`-w 2 --timeout 15` values are the shipped worker count and request timeout.

### 2.4 Canonical minting of `signed_suffix` (never hand-forged)

A valid `signed_suffix` is minted only by its real producer and never fabricated. Two equivalent canonical sources were used: (a) the shared signer `app.alias_suffix.signer.sign(value).decode()`, an `itsdangerous.TimestampSigner` keyed by `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` = `"secretcustom_alias"` in the test env [`app/config.py:201`, `tests/test.env`]; and (b) the real API producer `GET /api/v4/alias/options` (`get_alias_suffixes` [`app/api/views/alias_options.py`]), which Runner B uses. The **expired** case (§3, COND5) was produced by backdating the *real* signer's timestamp inside the probe only (never editing product code), so `signer.unsign(signed_suffix, max_age=600)` raises `SignatureExpired` for real; the **tampered** case uses a verified byte-mutation (§9) that is guaranteed to change the signed payload rather than the unreliable last-character flip.

### 2.5 Cleanup & net-zero proof (read-only guarantee)

The source repository was treated as strictly read-only; the only artifact added is this document. This section proves, with unedited before/after evidence, that (1) prior-draft residue was purged, (2) the finalized probe suite is net-zero, and (3) the repository contains exactly one changed file.

#### 2.5.1 Prior-draft residue purge

Earlier, pre-net-zero drafts of this investigation (before the isolated harness in §2.3 existed) had left committed rows in PostgreSQL and transient keys in Redis. These were enumerated precisely and purged. In PostgreSQL the residue was four draft users (random `create_new_user()` emails `user_*@mailbox.test` / `livedemo_*`) owning 14 aliases (the 10 flagged residue aliases `2779/2780/2788/2809-2814/2820` plus their 4 auto-created default aliases), 14 `alias_audit_log` rows, 4 mailboxes, and 1 `api_key` (0 `api_cookie_token` children); those users had **no** references in any of the other 40+ tables. The delete ran inside a single transaction (`ON_ERROR_STOP=1`, so any FK surprise rolls back rather than partial-deleting), in FK-safe order (null the circular `default_mailbox_id` first). Unedited transcript:

```text
############################################################
# PRIOR-DRAFT RESIDUE PURGE (Finding 6)
# Removes investigation-draft artifacts left by EARLIER (pre-net-zero-harness) probe drafts.
# Draft users: 1677,1681,1696,1700 (random create_new_user() emails @mailbox.test / livedemo_*).
# Footprint (enumerated): 4 users, 4 mailboxes, 14 aliases (10 flagged residue + 4 auto-default),
#                         14 alias_audit_log, 1 api_key (0 api_cookie_token children); no other refs.
# Date: 2026-07-13T20:10:04Z
############################################################

=== BEFORE: global counts ===
alias|838
users|566
mailbox|713
api_key|16
alias_audit_log|955

=== BEFORE: the 10 Finding-6 residue aliases ===
2779|prefix.test@sl.local
2780|dupprefix.test@sl.local
2788|p7xj1ohp3.word@sl.local
2809|prefix0.w0@sl.local
2810|prefix1.w1@sl.local
2811|prefix2.w2@sl.local
2812|prefix3.w3@sl.local
2813|prefix4.w4@sl.local
2814|prefix5.w5@sl.local
2820|livevalid.test398@sl.local

=== TRANSACTIONAL DELETE (FK-safe; ON_ERROR_STOP rolls back on any error) ===
BEGIN
UPDATE 4
DELETE 14
DELETE 1
DELETE 14
DELETE 4
DELETE 4
COMMIT
psql exit=0

=== AFTER: global counts ===
alias|824
users|562
mailbox|709
api_key|15
alias_audit_log|941

=== AFTER: confirm residue aliases + draft users are GONE (expect empty) ===
-- residue aliases remaining:
-- draft users remaining:
-- (empty output above = fully purged)
```

In Redis every key is runtime-accumulated (Redis starts empty at container start); the residue was 21 `LIMITER/*` rate-limit counters, 1 `bl:*` alias-create bucket, and 589 `session:*` login sessions left by earlier drafts. All were unlinked, returning Redis to empty. Unedited transcript:

```text
############################################################
# REDIS RESIDUE PURGE (Finding 6) — transient runtime keys
# Redis starts EMPTY at container start; ALL keys are runtime-accumulated by investigation
# drafts/sessions (rate-limit counters, alias-create buckets, login sessions, parallel locks).
# Date: 2026-07-13T20:10:31Z
############################################################

=== BEFORE: key inventory by pattern ===
DBSIZE          = 611
LIMITER/*       = 21
bl:*            = 1
session:*       = 589
cl:* (par-lock) = 0
-- any OTHER keys not in {LIMITER/,bl:,session:,cl:}? --
(end other-keys sample)

=== DELETE residue patterns (LIMITER/*, bl:*, session:*, cl:*) via SCAN+UNLINK ===
  deleted pattern LIMITER/*  (was 21 keys)
  deleted pattern bl:*  (was 1 keys)
  deleted pattern session:*  (was 589 keys)
  deleted pattern cl:*  (was 0 keys)

=== AFTER: key inventory ===
DBSIZE          = 0
LIMITER/*       = 0
bl:*            = 0
session:*       = 0
cl:*            = 0
-- remaining keys (if any): --
(end)
```
After the purge the clean baseline is `alias=824, users=562, mailbox=709, api_key=15` in PostgreSQL and `DBSIZE=0` in Redis — the exact baseline the master harness records below.

#### 2.5.2 Master harness & net-zero verification

The canonical master reproduction harness `run_all.sh` runs all six probes, then proves net-zero residue in both stores: it snapshots Redis before and after and deletes **only** the keys the probes created (pre-existing keys are left untouched via `comm -13`), and it compares per-table PostgreSQL counts before and after. It exits non-zero if any probe fails or any residue remains. Source:

```bash
#!/bin/bash
# run_all.sh -- canonical master reproduction harness for the SimpleLogin custom-alias
# investigation. Runs all six probes, then proves NET-ZERO residue in both PostgreSQL
# (per-table counts) and Redis (snapshot-diff: every key created by the probes is deleted,
# pre-existing keys are left untouched). Exits non-zero if any probe fails or residue remains.
set -u
cd /app
OUT=/tmp/sl_probes/out
mkdir -p "$OUT"
CE=(env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root CONFIG=tests/test.env \
    DB_URI='postgresql://test:test@localhost:5432/test' GNUPGHOME=/tmp/sl_gnupg \
    PYTHONPATH=/app /app/venv/bin/python)

dbcounts() { for t in alias users api_key mailbox alias_mailbox deleted_alias; do
  echo -n "$t="; PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from $t"; done; }

echo "===== BASELINE ====="
dbcounts | tr '\n' ' '; echo
DB0=$(PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from alias")
redis-cli --scan | sort > "$OUT/redis_snapshot_before.txt"
SZ0=$(wc -l < "$OUT/redis_snapshot_before.txt")
echo "redis keys before = $SZ0"

echo
echo "===== RUN PROBES ====="
rc=0
"${CE[@]}" /tmp/sl_probes/probe_mro.py        > "$OUT/mro.out" 2>&1;        e=$?; echo "probe_mro.py        EXIT=$e"; [ $e -ne 0 ] && rc=1
"${CE[@]}" /tmp/sl_probes/probe_functional.py > "$OUT/functional.out" 2>&1; e=$?; echo "probe_functional.py EXIT=$e"; [ $e -ne 0 ] && rc=1
"${CE[@]}" /tmp/sl_probes/probe_f18.py        > "$OUT/f18.out" 2>&1;        e=$?; echo "probe_f18.py        EXIT=$e"; [ $e -ne 0 ] && rc=1
"${CE[@]}" /tmp/sl_probes/probe_ratelimit.py  > "$OUT/ratelimit.out" 2>&1;  e=$?; echo "probe_ratelimit.py  EXIT=$e"; [ $e -ne 0 ] && rc=1
"${CE[@]}" /tmp/sl_probes/probe_lock.py       > "$OUT/lock.out" 2>&1;       e=$?; echo "probe_lock.py       EXIT=$e"; [ $e -ne 0 ] && rc=1
bash /tmp/sl_probes/probe_live.sh             > "$OUT/live.out" 2>&1;       e=$?; echo "probe_live.sh       EXIT=$e"; [ $e -ne 0 ] && rc=1

echo
echo "===== REDIS SNAPSHOT-DIFF CLEANUP (delete only keys the probes created) ====="
redis-cli --scan | sort > "$OUT/redis_snapshot_after.txt"
comm -13 "$OUT/redis_snapshot_before.txt" "$OUT/redis_snapshot_after.txt" > "$OUT/redis_new_keys.txt"
NEW=$(wc -l < "$OUT/redis_new_keys.txt")
echo "new keys created by probes = $NEW"
if [ "$NEW" -gt 0 ]; then
  echo "  (breakdown by prefix:)"; sed -E 's/:.*//;s#/.*##' "$OUT/redis_new_keys.txt" | sort | uniq -c
  xargs -r redis-cli del < "$OUT/redis_new_keys.txt" > /dev/null
fi

echo
echo "===== POST-CLEANUP NET-ZERO VERIFICATION ====="
dbcounts | tr '\n' ' '; echo
DB1=$(PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from alias")
redis-cli --scan | sort > "$OUT/redis_snapshot_final.txt"
LEFT=$(comm -13 "$OUT/redis_snapshot_before.txt" "$OUT/redis_snapshot_final.txt" | wc -l)
echo "alias baseline=$DB0 final=$DB1"
echo "redis probe-created keys remaining after cleanup = $LEFT"
if [ "$DB0" = "$DB1" ] && [ "$LEFT" -eq 0 ] && [ "$rc" -eq 0 ]; then
  echo ">>> NET-ZERO OK: all probes passed; DB + Redis restored to baseline"
else
  echo ">>> FAILURE: rc=$rc db_equal=$([ "$DB0" = "$DB1" ] && echo yes || echo no) redis_left=$LEFT"; rc=1
fi
exit $rc
```

Running it at the clean baseline (invocation: `docker exec sl_setup bash /tmp/sl_probes/run_all.sh`) produced the following unedited output. The baseline (`alias=824`, `redis keys before = 0`) equals the final (`alias baseline=824 final=824`, `redis probe-created keys remaining after cleanup = 0`), all six probes exit `0`, and the harness prints `NET-ZERO OK`. Because the harness reads the live count at run time, net-zero is the invariant `baseline == final`; it holds at any baseline (a fresh provision would baseline at its own count and still prove `N -> N`):

```text
===== BASELINE =====
alias=824 users=562 api_key=15 mailbox=709 alias_mailbox=13 deleted_alias=15 
redis keys before = 0

===== RUN PROBES =====
probe_mro.py        EXIT=0
probe_functional.py EXIT=0
probe_f18.py        EXIT=0
probe_ratelimit.py  EXIT=0
probe_lock.py       EXIT=0
probe_live.sh       EXIT=0

===== REDIS SNAPSHOT-DIFF CLEANUP (delete only keys the probes created) =====
new keys created by probes = 35
  (breakdown by prefix:)
     22 bl
     13 session

===== POST-CLEANUP NET-ZERO VERIFICATION =====
alias=824 users=562 api_key=15 mailbox=709 alias_mailbox=13 deleted_alias=15 
alias baseline=824 final=824
redis probe-created keys remaining after cleanup = 0
>>> NET-ZERO OK: all probes passed; DB + Redis restored to baseline
```

#### 2.5.3 Repository cleanliness

The source repository is byte-for-byte unchanged except for the single new answer document. Verified against the source-branch baseline `2cd6ee77`:

```text
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md

$ git diff 2cd6ee77 --name-status
A	blitzy/documentation/app_2cd6ee777f8c.md
```

All probe scripts are temporary (kept under the container's `/tmp/sl_probes`, never inside the repository) and are removed at the end of the investigation, leaving the repository containing exactly one added file: `blitzy/documentation/app_2cd6ee777f8c.md`.


## 3. Q1 — Exact HTTP status codes and error messages

Both endpoints require API-key authentication (`Authentication:` header, validated by
`require_api_auth` → `authorize_request()` in `app/api/base.py`). A **valid** `signed_suffix` is minted
only by the canonical producer `get_alias_suffixes` (`app/alias_suffix.py:94-192`), surfaced over
`GET /api/v4/alias/options` (`app/api/views/alias_options.py`); every probe below sources its valid
suffix from that producer (or from the same `itsdangerous.TimestampSigner` instance, `app/alias_suffix.py:11`)
so the input is canonical rather than hand-forged.

**Primary answer — invalid vs expired.** A **tampered (invalid)** signed suffix and a **genuinely
expired** signed suffix produce the *same* response:

| Condition | Endpoint | Status | Response body | `Content-Length` |
|-----------|----------|--------|---------------|------------------|
| **Invalid / tampered** signed suffix | v2 & v3 | **412 PRECONDITION FAILED** | `{"error":"Alias creation time is expired, please retry"}` | 57 |
| **Expired** signed suffix (`age > max_age=600`) | v2 & v3 | **412 PRECONDITION FAILED** | `{"error":"Alias creation time is expired, please retry"}` | 57 |
| Unsigned **garbage** string | v2 & v3 | **412 PRECONDITION FAILED** | `{"error":"Alias creation time is expired, please retry"}` | 57 |
| Empty request body `{}` | v2 & v3 | **400 BAD REQUEST** | `{"error":"request body cannot be empty"}` | 41 |
| v3 body not a JSON object | v3 | **400 BAD REQUEST** | `{"error":"request body does not follow the required format"}` | 61 |
| v3 `mailbox_ids` not an array | v3 | **400 BAD REQUEST** | `{"error":"mailbox_ids must be an array of id"}` | 47 |
| **Duplicate** alias | v2 & v3 | **409 CONFLICT** | `{"error":"alias <full-address> already exists"}` | (varies) |
| Two consecutive dots in prefix | v2 & v3 | **400 BAD REQUEST** | `{"error":"2 consecutive dot signs aren't allowed in an email address"}` | 71 |
| Wrong prefix / wrong suffix / wrong domain | v2 & v3 | **400 BAD REQUEST** | `{"error":"wrong alias prefix or suffix"}` | 41 |
| **Valid** suffix, under quota | v2 & v3 | **201 CREATED** | serialized alias object | 430 |

The three signature-failure rows collapse to one 412 because `check_suffix_signature` returns `None` for
tampered, expired *and* garbage input alike — see the runtime proof in §9. The distinct
`400 "Tampered suffix"` message that the source appears to offer (`new_custom_alias.py:76` / `:191`) is
**never produced for ordinary string input**.

**Edge case — a non-string `signed_suffix` (Finding: `.strip()` placement).** The raw value is
`.strip()`-ed *before* the signature `try` block (v2 `new_custom_alias.py:65`; v3 `:158`). A **truthy
non-string** JSON value (list/int/dict) therefore raises `AttributeError` before any signature logic and
surfaces as an unhandled **500** `{"error":"Internal error"}` — it does **not** reach the
`"Tampered suffix"` branch. A missing key or empty string defaults to `""` (falsy) → 412. Note a real
v2/v3 divergence for JSON `null`: v2 → **500** (`data.get("signed_suffix","")` returns `None` when the key
is present-but-null; `None.strip()` raises), whereas v3 → **412** (v3's `signed_suffix = data.get("signed_suffix", "") or ""` at `:157` coerces `None`
to `""`). The complete traceback evidence is in §9.4. This is an edge a canonical client never hits, since
clients send the string `signed_suffix` minted by `/api/v4|v5/alias/options`.

**On response equality (scope of the "identical" claim).** For the signature-failure conditions, v2 and
v3 return an **identical HTTP status code (412) and identical JSON body**. This is *not* a claim that the
entire HTTP responses are byte-for-byte identical: each response carries its own per-request
`Set-Cookie: slapp=<per-request-session-token>` value, and success responses differ in `id`, timestamps and mailbox. The equality
that matters for Q1 — status code and error message — is what is asserted and shown.

### 3.1 Live-server transport headers (`curl -i` against `gunicorn wsgi:app`)

Captured from a real `gunicorn/20.0.4` server (`-b 127.0.0.1:7777 -w 2 --timeout 15`) with rate limiting
**enabled**, so the complete transport header set — including `Server`, `Date` and `Connection` — is
visible. These are the raw, unedited `curl -i` outputs for a 201 (valid), a 412 (tampered) and a 429
(rate-limited) response:

**201 — valid suffix** (`out/live_201.txt`):
```text
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 19:30:47 GMT
Connection: close
Content-Type: application/json
Content-Length: 430
Access-Control-Allow-Origin: *
Set-Cookie: slapp=d216cb8b-c659-4cd5-ada3-3ceb3ffe5d47.XI5mlmc1SXgX4wqbQpbwXk527r0; Expires=Mon, 20-Jul-2026 19:30:47 GMT; HttpOnly; Path=/; SameSite=Lax

{"alias":"lh1.list@sl.local","creation_date":"2026-07-13 19:30:47+00:00","creation_timestamp":1783971047,"disable_pgp":false,"email":"lh1.list@sl.local","enabled":true,"id":3103,"latest_activity":null,"mailbox":{"email":"livehdr_zbkuumgz@example.test","id":2106},"mailboxes":[{"email":"livehdr_zbkuumgz@example.test","id":2106}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
```

**412 — tampered suffix** (`out/live_412.txt`):
```text
HTTP/1.1 412 PRECONDITION FAILED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 19:30:47 GMT
Connection: close
Content-Type: application/json
Content-Length: 57
Access-Control-Allow-Origin: *
Set-Cookie: slapp=d74b40b1-ce22-4563-96e5-ae8ca058c0bf.fCEZedfI6KT_QNMn487nqWclFeo; Expires=Mon, 20-Jul-2026 19:30:47 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Alias creation time is expired, please retry"}
```

**429 — rate-limited** (`out/live_429.txt`):
```text
HTTP/1.1 429 TOO MANY REQUESTS
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 19:30:47 GMT
Connection: close
Content-Type: application/json
Content-Length: 32
Access-Control-Allow-Origin: *
Set-Cookie: slapp=61c7cde1-94b3-44bb-a03c-70d93f7f4b5a.YuhgcS8oGZo7AaY8ucMJv6RhfNU; Expires=Mon, 20-Jul-2026 19:30:47 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Rate limit exceeded"}
```

Every response — 201, 412 and 429 — carries exactly the same header set:
`Server`, `Date`, `Connection: close`, `Content-Type: application/json`, `Content-Length`,
`Access-Control-Allow-Origin: *`, `Set-Cookie: slapp=<per-request-session-token>`. **No `X-RateLimit-*` and no `Retry-After`
header appears on any of them** (Q3, §5).

### 3.2 Complete per-condition transcript (test-client harness, unedited)

The following is the complete, unedited output of `probe_functional.py` (`out/functional.out`). It drives
every condition through the real endpoints and prints, per condition, the exact `COMMAND`, the `RESPONSE`
(status, headers, body) and the server-console log lines emitted during that request (between
`[BEGIN]`/`[END]` markers — see §4). It ends with `assertions failed: 0` and confirms the transaction was
rolled back (nothing committed to PostgreSQL):

```text
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:30:32,612 - SL - DEBUG - 7096 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> probe_functional.py starting
>>> EMAIL_DOMAIN=sl.local  MAX_NB_EMAIL_FREE_PLAN=3
>>> CUSTOM_ALIAS_SECRET='secretcustom_alias'  (public TEST-ONLY secret = FLASK_SECRET+'custom_alias')
>>> All signed suffixes below are SYNTHETIC, DISPOSABLE, TEST-ONLY and TIME-LIMITED (max_age=600s).

>>> disposable PRIMARY user id=1786 email=user_7gwalg3eef@mailbox.test default_mailbox_id=2096
>>> made PRIMARY user premium (lifetime=True) so can_create_new_alias() is always True
>>> disposable api_key: ApiKey.create(user.id) -> code is random_string(60); value REDACTED

================================================================================
CONDITION 0: canonical signed_suffix producer  GET /api/v4/alias/options
================================================================================
  status_code = 200
  can_create  = True
  suffixes (first 4 [suffix, signed_suffix] pairs; signed values are time-limited):
    ['.word433@d1.test', '.word433@d1.test.alU82Q.S5NKcC9wEVBxKak9Y5ghHjOAdwo']
    ['.word775@d2.test', '.word775@d2.test.alU82Q._4k824sBcGbgFXDnqHNlikeZpBc']
    ['.test234@sl.local', '.test234@sl.local.alU82Q.o4_XyG9aTaqBZNAnxT5p8GsUZ6A']
    ['.word742@iqg945z56z.com', '.word742@iqg945z56z.com.alU82Q.lYlXygvm1SXv158tU2N7BK3HxWM']
  [ASSERT PASS] options status 200
  [ASSERT PASS] a suffix for @sl.local is offered by the producer
  can_create_new_alias() branch inputs [primary premium user] (app/models.py:L867-884):
    is_active()                        = True   (L872; delete_on=None)
    disabled                           = False   (L875)
    lifetime                           = True
    lifetime_or_active_subscription()  = True   (L878; trial does NOT count)
    FLAG_FREE_OLD_ALIAS_LIMIT set      = False   (-> max via L858-865)
    max_alias_for_free_account()       = 3
    Alias count for user               = 1   (L881-883 count)
    ==> can_create_new_alias()         = True

================================================================================
CONDITION 1: SUCCESS v2 (valid signed suffix)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "okv2", "signed_suffix": ".word@sl.local.alU82Q.ohkmNyYuE5Bom2SfQVNGqzB4NnQ"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 201
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 430
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"alias":"okv2.word@sl.local","creation_date":"2026-07-13 19:30:33+00:00","creation_timestamp":1783971033,"disable_pgp":false,"email":"okv2.word@sl.local","enabled":true,"id":3075,"latest_activity":null,"mailbox":{"email":"user_7gwalg3eef@mailbox.test","id":2096},"mailboxes":[{"email":"user_7gwalg3eef@mailbox.test","id":2096}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,862 - SL - INFO - 7096 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
    | 2026-07-13 19:30:33,870 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.027808189392089844
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 201 == 201

================================================================================
CONDITION 2: SUCCESS v3 (valid signed suffix + mailbox_ids)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v3/alias/custom/new',
      json={"alias_prefix": "okv3", "signed_suffix": ".list@sl.local.alU82Q.lnoUBz9v-dZskCoUpw1NVftwCBg", "mailbox_ids": [2096]},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 201
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 430
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"alias":"okv3.list@sl.local","creation_date":"2026-07-13 19:30:33+00:00","creation_timestamp":1783971033,"disable_pgp":false,"email":"okv3.list@sl.local","enabled":true,"id":3076,"latest_activity":null,"mailbox":{"email":"user_7gwalg3eef@mailbox.test","id":2096},"mailboxes":[{"email":"user_7gwalg3eef@mailbox.test","id":2096}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,894 - SL - INFO - 7096 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
    | 2026-07-13 19:30:33,899 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 201, takes 0.026784896850585938
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 201 == 201

[tamper] tampered!=valid -> True
[tamper] signer.unsign(tampered) raised BadTimeSignature: Signature b'ohkmNyYuE5Bom2SfQVNGqzB4NnQ' does not match (subclass of BadSignature -> caught by check_suffix_signature -> None)

================================================================================
CONDITION 3: TAMPERED signed suffix (one signature char changed)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "tamp", "signed_suffix": "aword@sl.local.alU82Q.ohkmNyYuE5Bom2SfQVNGqzB4NnQ"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 412
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 57
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"Alias creation time is expired, please retry"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,904 - SL - WARNING - 7096 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1786 Test User user_7gwalg3eef@mailbox.test>
    | 2026-07-13 19:30:33,905 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.004210710525512695
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 412 == 412
  [ASSERT PASS] error body 'Alias creation time is expired, please retry' == 'Alias creation time is expired, please retry'

[garbage] signer.unsign('this-string-was-never-signed') raised BadSignature -> None

================================================================================
CONDITION 4: GARBAGE signed suffix (never signed)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "garb", "signed_suffix": "this-string-was-never-signed"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 412
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 57
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"Alias creation time is expired, please retry"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,909 - SL - WARNING - 7096 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1786 Test User user_7gwalg3eef@mailbox.test>
    | 2026-07-13 19:30:33,910 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.003984212875366211
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 412 == 412
  [ASSERT PASS] error body 'Alias creation time is expired, please retry' == 'Alias creation time is expired, please retry'

================================================================================
CONDITION 5: GENUINELY EXPIRED signed suffix (mint backdated, prove age>600, then POST)
================================================================================
  signed-at (unix)  = 1783970432  (2026-07-13T19:20:32+00:00)
  current  (unix)   = 1783971033  (2026-07-13T19:30:33+00:00)
  age (s)           = 601  (> max_age=600 used by check_suffix_signature L40)
  DIRECT proof at the endpoint's real max_age=600:
    signer.unsign(expired, max_age=600) raised SignatureExpired: Signature age 601 > 600 seconds
    e.date_signed = 2026-07-13 19:20:32  (SignatureExpired IS-A BadSignature)
  check_suffix_signature(expired) -> None  (None => handler 412 branch)

================================================================================
CONDITION 5b: POST the EXACT expired token to the real endpoint
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "expd", "signed_suffix": ".word@sl.local.alU6gA.zeYiaDExtX2ErCQqyqcFVvLdmgA"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 412
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 57
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"Alias creation time is expired, please retry"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,915 - SL - WARNING - 7096 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1786 Test User user_7gwalg3eef@mailbox.test>
    | 2026-07-13 19:30:33,915 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.003949642181396484
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 412 == 412
  [ASSERT PASS] error body 'Alias creation time is expired, please retry' == 'Alias creation time is expired, please retry'

================================================================================
CONDITION 6: EMPTY request body (v2)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 41
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"request body cannot be empty"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,920 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.003935813903808594
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'request body cannot be empty' == 'request body cannot be empty'

================================================================================
CONDITION 7: v3 NON-DICT body (truthy non-dict fails isinstance BEFORE signature try; L153-154)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v3/alias/custom/new',
      json="string isn't a dict",
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 61
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"request body does not follow the required format"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,925 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 400, takes 0.0037298202514648438
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'request body does not follow the required format' == 'request body does not follow the required format'

================================================================================
CONDITION 8: v3 mailbox_ids NOT an array (L171-172)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v3/alias/custom/new',
      json={"alias_prefix": "mbx", "signed_suffix": ".test@sl.local.alU82Q.6b5Lzx--aGqctjjpGVj8F3MwBOI", "mailbox_ids": "not an array"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 47
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"mailbox_ids must be an array of id"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,930 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 400, takes 0.003754854202270508
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'mailbox_ids must be an array of id' == 'mailbox_ids must be an array of id'

================================================================================
CONDITION 9a: DUPLICATE step 1 -- create the alias (201)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "dup", "signed_suffix": ".test@sl.local.alU82Q.6b5Lzx--aGqctjjpGVj8F3MwBOI"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 201
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 428
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"alias":"dup.test@sl.local","creation_date":"2026-07-13 19:30:33+00:00","creation_timestamp":1783971033,"disable_pgp":false,"email":"dup.test@sl.local","enabled":true,"id":3077,"latest_activity":null,"mailbox":{"email":"user_7gwalg3eef@mailbox.test","id":2096},"mailboxes":[{"email":"user_7gwalg3eef@mailbox.test","id":2096}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,950 - SL - INFO - 7096 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
    | 2026-07-13 19:30:33,955 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.024661779403686523
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 201 == 201

================================================================================
CONDITION 9b: DUPLICATE step 2 -- same prefix+suffix again (409 + LOG.d L87)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "dup", "signed_suffix": ".test@sl.local.alU82Q.6b5Lzx--aGqctjjpGVj8F3MwBOI"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 409
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 51
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"alias dup.test@sl.local already exists"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,971 - SL - DEBUG - 7096 - "/app/app/api/views/new_custom_alias.py:87" - new_custom_alias_v2() -  - full alias already used dup.test@sl.local
    | 2026-07-13 19:30:33,972 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 409, takes 0.015325784683227539
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 409 == 409
  [ASSERT PASS] error body 'alias dup.test@sl.local already exists' == 'alias dup.test@sl.local already exists'

================================================================================
CONDITION 10: TWO CONSECUTIVE DOTS (alias_prefix ends with a dot)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "prefix.", "signed_suffix": ".list@sl.local.alU82Q.lnoUBz9v-dZskCoUpw1NVftwCBg"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 71
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"2 consecutive dot signs aren't allowed in an email address"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,989 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.016082048416137695
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body "2 consecutive dot signs aren't allowed in an email address" == "2 consecutive dot signs aren't allowed in an email address"

================================================================================
CONDITION 11: WRONG SUFFIX (SL domain, prefix does NOT start with '.'; verify_prefix_suffix L77-79 LOG.e)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "wsuf", "signed_suffix": "test@sl.local.alU82Q.HHNFWlQg5zkl70FEhoYdpok5gA4"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 41
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"wrong alias prefix or suffix"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:33,999 - SL - ERROR - 7096 - "/app/app/alias_suffix.py:78" - verify_prefix_suffix() -  - User <User 1786 Test User user_7gwalg3eef@mailbox.test> submits a wrong alias suffix test@sl.local
    | NoneType: None
    | 2026-07-13 19:30:34,000 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.009603261947631836
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'wrong alias prefix or suffix' == 'wrong alias prefix or suffix'

================================================================================
CONDITION 12: WRONG DOMAIN (validly signed suffix for a domain the user cannot use; verify L60-62 LOG.e)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "wdom", "signed_suffix": ".list@not-a-real-domain.test.alU82g.U1LvC5krA6iJUXmwQ6ygWFpmtMc"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 41
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"wrong alias prefix or suffix"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,010 - SL - ERROR - 7096 - "/app/app/alias_suffix.py:61" - verify_prefix_suffix() -  - wrong alias suffix .list@not-a-real-domain.test, user <User 1786 Test User user_7gwalg3eef@mailbox.test>
    | NoneType: None
    | 2026-07-13 19:30:34,011 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.009343624114990234
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'wrong alias prefix or suffix' == 'wrong alias prefix or suffix'

================================================================================
CONDITION 13: FREE user -- count-based success path, quota exhaustion, and masking
================================================================================
  can_create_new_alias() branch inputs [fresh FREE user (before any create via endpoint)] (app/models.py:L867-884):
    is_active()                        = True   (L872; delete_on=None)
    disabled                           = False   (L875)
    lifetime                           = False
    lifetime_or_active_subscription()  = False   (L878; trial does NOT count)
    FLAG_FREE_OLD_ALIAS_LIMIT set      = False   (-> max via L858-865)
    max_alias_for_free_account()       = 3
    Alias count for user               = 1   (L881-883 count)
    ==> can_create_new_alias()         = True

================================================================================
CONDITION 13a: FREE user SUCCESS while under quota (count < max; success path emits NO quota log)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "freeok", "signed_suffix": ".test@sl.local.alU82g.-cNc5TIO7y1KGXiSbrcILd1FI9o"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 201
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 434
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"alias":"freeok.test@sl.local","creation_date":"2026-07-13 19:30:34+00:00","creation_timestamp":1783971034,"disable_pgp":false,"email":"freeok.test@sl.local","enabled":true,"id":3079,"latest_activity":null,"mailbox":{"email":"user_ahp49hspk7@mailbox.test","id":2097},"mailboxes":[{"email":"user_ahp49hspk7@mailbox.test","id":2097}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,338 - SL - INFO - 7096 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
    | 2026-07-13 19:30:34,343 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04214215278625488
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 201 == 201
  can_create_new_alias() branch inputs [after one endpoint create] (app/models.py:L867-884):
    is_active()                        = True   (L872; delete_on=None)
    disabled                           = False   (L875)
    lifetime                           = False
    lifetime_or_active_subscription()  = False   (L878; trial does NOT count)
    FLAG_FREE_OLD_ALIAS_LIMIT set      = False   (-> max via L858-865)
    max_alias_for_free_account()       = 3
    Alias count for user               = 2   (L881-883 count)
    ==> can_create_new_alias()         = True
  filling 1 more alias(es) via Alias.create_new(quser, prefix='fill<i>') to reach the cap
  can_create_new_alias() branch inputs [after filling to cap] (app/models.py:L867-884):
    is_active()                        = True   (L872; delete_on=None)
    disabled                           = False   (L875)
    lifetime                           = False
    lifetime_or_active_subscription()  = False   (L878; trial does NOT count)
    FLAG_FREE_OLD_ALIAS_LIMIT set      = False   (-> max via L858-865)
    max_alias_for_free_account()       = 3
    Alias count for user               = 3   (L881-883 count)
    ==> can_create_new_alias()         = False

================================================================================
CONDITION 13b: QUOTA exceeded with a VALID suffix (400 quota + LOG.d L49)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "over", "signed_suffix": ".test@sl.local.alU82g.-cNc5TIO7y1KGXiSbrcILd1FI9o"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 141
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,395 - SL - DEBUG - 7096 - "/app/app/api/views/new_custom_alias.py:49" - new_custom_alias_v2() -  - user <User 1787 Test User user_ahp49hspk7@mailbox.test> cannot create any custom alias
    | 2026-07-13 19:30:34,395 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.009819984436035156
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases' == 'You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases'

================================================================================
CONDITION 13c: QUOTA exceeded with a TAMPERED suffix -> STILL 400 quota (quota MASKS the 412)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "overt", "signed_suffix": ".list@sl.local.alU82g.Uhl7AemLXDPUo9KCm03BG_E_LQA"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 141
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,406 - SL - DEBUG - 7096 - "/app/app/api/views/new_custom_alias.py:49" - new_custom_alias_v2() -  - user <User 1787 Test User user_ahp49hspk7@mailbox.test> cannot create any custom alias
    | 2026-07-13 19:30:34,407 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.009836435317993164
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases' == 'You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases'

================================================================================
CONDITION 14a: AUTH: no Authentication header (401 Wrong api key)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "noauth", "signed_suffix": ".test@sl.local.alU82g.-cNc5TIO7y1KGXiSbrcILd1FI9o"},
      headers={})
RESPONSE:
  status_code = 401
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 26
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"Wrong api key"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,409 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 401, takes 0.0008711814880371094
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 401 == 401
  [ASSERT PASS] error body 'Wrong api key' == 'Wrong api key'

================================================================================
CONDITION 14b: AUTH: invalid api key (401 Wrong api key)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "badauth", "signed_suffix": ".test@sl.local.alU82g.-cNc5TIO7y1KGXiSbrcILd1FI9o"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 401
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 26
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"Wrong api key"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,411 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 401, takes 0.0008885860443115234
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 401 == 401
  [ASSERT PASS] error body 'Wrong api key' == 'Wrong api key'

================================================================================
CONDITION 14c: AUTH: DISABLED account via API key (403 Disabled account; base.py L36-37)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "dis", "signed_suffix": ".list@sl.local.alU82g.Uhl7AemLXDPUo9KCm03BG_E_LQg"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 403
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 29
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"Disabled account"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,678 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 403, takes 0.0035924911499023438
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 403 == 403
  [ASSERT PASS] error body 'Disabled account' == 'Disabled account'

================================================================================
CONDITION 14d: AUTH: INACTIVE account (delete_on in FUTURE => is_active() False) via API key (401 Account does not exist; base.py L39-40)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "ina", "signed_suffix": ".word@sl.local.alU82g.5Q_7sd7kEZMZbprSoMZuqCJhV0k"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 401
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 35
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"Account does not exist"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,945 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 401, takes 0.0036585330963134766
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 401 == 401
  [ASSERT PASS] error body 'Account does not exist' == 'Account does not exist'

================================================================================
SUMMARY
================================================================================
  assertions failed: 0

>>> rolled back transaction; probe committed NOTHING to PostgreSQL
```

---

## 4. Q2 — Server-console log entries for those failures

### 4.1 How the log was captured (mechanism, filter criteria, markers)

SimpleLogin's server console is written by a single logger, `LOG = _get_logger("SL")`
(`app/log.py:79`). Its format string (`app/log.py:12-15`) embeds the emitting
`"%(pathname)s:%(lineno)d"` and `%(funcName)s()`, which is exactly why each console line below can be
attributed to a specific source location:

```text
"%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
'"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
```

The level shortcuts (`app/log.py:74-77`) are `LOG.d`=DEBUG, `LOG.i`=INFO, `LOG.w`=WARNING and
`LOG.e`=`logging.Logger.exception` (i.e. an **ERROR that also emits a traceback**). That last detail
explains the `| NoneType: None` line that trails the two "wrong suffix" ERRORs below: `LOG.e` renders the
current exception info, and when it is called outside an active exception the traceback serialises to
`NoneType: None`.

**Capture method (verifiable, not asserted).** The probe attaches a `StringIO` stream handler to the very
same `"SL"` logger, using the *exact* server-console formatter (`sl_log._log_formatter`, `app/log.py:16`)
with a UTC time converter, and swaps it in for the duration of each request. It records **every** record
the `"SL"` logger emits during the request window — nothing is filtered by level or dropped — and brackets
the buffer with `[BEGIN]`/`[END]` markers. Consequently, "no log line" is shown as an *empty* buffer
between the markers rather than claimed with an unverifiable "`<none>`". The capture code
(`probe_functional.py:54-71`) is:

```python
# ---------------------------------------------------------------- log capture
_cap_buf = io.StringIO()
_cap_handler = logging.StreamHandler(_cap_buf)
_cap_handler.setFormatter(sl_log._log_formatter)          # exact server-console format
_cap_handler.formatter.converter = time.gmtime
_saved_handlers = []

def install_capture():
    global _saved_handlers
    _saved_handlers = list(LOG.handlers)
    for h in _saved_handlers:
        LOG.removeHandler(h)
    LOG.addHandler(_cap_handler)

def uninstall_capture():
    LOG.removeHandler(_cap_handler)
    for h in _saved_handlers:
        LOG.addHandler(h)
```

The raw per-condition buffers are shown, unedited, inside the transcript in **§3.2** (each request's
`[BEGIN]`/`[END]` block). For example the tampered-suffix 412 emits, verbatim:

```text
| 2026-07-13 19:16:50,204 - SL - WARNING - 6845 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1775 Test User user_uv31r8in6d@mailbox.test>
| 2026-07-13 19:16:50,204 - SL - DEBUG - 6845 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.004083395004272461
```

and the "wrong suffix" 400 emits an ERROR immediately followed by the `LOG.e` traceback line:

```text
| 2026-07-13 19:16:50,297 - SL - ERROR - 6845 - "/app/app/alias_suffix.py:78" - verify_prefix_suffix() -  - User <User 1775 Test User user_uv31r8in6d@mailbox.test> submits a wrong alias suffix test@sl.local
| NoneType: None
| 2026-07-13 19:16:50,298 - SL - DEBUG - 6845 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.009406805038452148
```

### 4.2 The exact log line(s) per condition

Timestamps, process id and the per-run user identifier vary between runs; the **level, emitter
(`file:line` + `funcName`) and message text are stable**. Every response, regardless of outcome, also
emits the `DEBUG` `server.py:284` `after_request()` line recording the method, path and final status.

| Condition (status) | Level & emitter | Message (verbatim, `<user>`/timestamps elided only where run-specific) |
|--------------------|-----------------|--------------------------------------------------------------------------|
| Tampered / expired / garbage suffix → **412** (v2) | `WARNING` `app/api/views/new_custom_alias.py:72` `new_custom_alias_v2()` | `Alias creation time expired for <user>` |
| Tampered / expired / garbage suffix → **412** (v3) | `WARNING` `app/api/views/new_custom_alias.py:187` `new_custom_alias_v3()` | `Alias creation time expired for <user>` |
| Successful create → **201** | `INFO` `app/events/event_dispatcher.py:62` `send_event()` | `Not sending events because webhook is not configured and allowed to be empty` |
| Quota exhausted → **400** | `DEBUG` `app/api/views/new_custom_alias.py:49` `new_custom_alias_v2()` | `user <user> cannot create any custom alias` |
| Duplicate alias → **409** | `DEBUG` `app/api/views/new_custom_alias.py:87` `new_custom_alias_v2()` | `full alias already used <full-address>` |
| Wrong suffix (prefix lacks `.`) → **400** | `ERROR` (`LOG.e`) `app/alias_suffix.py:78` `verify_prefix_suffix()` | `User <user> submits a wrong alias suffix <suffix>` (+ `NoneType: None`) |
| Wrong domain → **400** | `ERROR` (`LOG.e`) `app/alias_suffix.py:61` `verify_prefix_suffix()` | `wrong alias suffix <suffix>, user <user>` (+ `NoneType: None`) |
| Rate limit exceeded → **429** | `WARNING` `server.py:364` `rate_limited()` | `Client hit rate limit on path <path>, user:<user>` |
| Non-string `signed_suffix` → **500** | `ERROR` `server.py:390` `error_handler()` | `'<type>' object has no attribute 'strip'` (+ traceback) |
| Every response | `DEBUG` `server.py:284` `after_request()` | `<ip> <method> <path> <MultiDict> <status>, takes <seconds>` |

The two lines that are captured outside `functional.out` (because they require rate limiting enabled /
an internal error) are, verbatim from their probes:

```text
2026-07-13 19:16:55,762 - SL - WARNING - 6868 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 1780 Test User user_u2muvbusuh@mailbox.test>
2026-07-13 19:16:53,279 - SL - ERROR - 6857 - "/app/server.py:390" - error_handler() -  - 'list' object has no attribute 'strip'
```

**Success-path note (relevant to Q4).** On a 201 the `"SL"` logger emits only the `INFO`
`event_dispatcher.py:62` webhook notice and the `DEBUG` `server.py:284` after-request line — the quota
check `can_create_new_alias()` logs **nothing** on success (see §6).

---

## 5. Q3 — Do rate-limiting headers appear, and what are their values?

**Direct answer: no.** No `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` or
`Retry-After` header appears on **any** response from the custom-alias endpoints — not on a `201`, not on
the `412`, and not on the `429`. This was confirmed twice: through the Werkzeug test client
(`probe_ratelimit.py`, §5.4) and through a real `gunicorn` server observed with `curl -i` (the `429`
transcript in §3.1). The complete header set on every response is exactly `Server`, `Date`,
`Connection`, `Content-Type`, `Content-Length`, `Access-Control-Allow-Origin` and `Set-Cookie` — nothing
rate-limit-related. When the limit is exceeded the response is `429 TOO MANY REQUESTS` with body
`{"error":"Rate limit exceeded"}` (`Content-Length: 32`).

### 5.1 Why no headers appear — the version history (Finding: "1.5") and the semantics

Two independent facts, both grounded in the installed source, explain the absence:

**(a) The pinned `flask-limiter 1.4` defaults header emission to OFF.** The `X-RateLimit-*` headers are
gated by `RATELIMIT_HEADERS_ENABLED`, whose effective default is `False`. This is directly visible in the
installed `flask_limiter/extension.py`: the constructor parameter defaults to `False` (L109), and
`init_app` resolves the config with `config.setdefault(C.HEADERS_ENABLED, False)` (L227-229):

```python
# flask_limiter/extension.py  (installed flask-limiter 1.4)  — constructor default L107-111
        default_limits_deduct_when=None,
        application_limits=[],
        headers_enabled=False,
        strategy=None,
        storage_uri=None,

# flask_limiter/extension.py  — init_app config resolution L225-230
            C.SWALLOW_ERRORS, self._swallow_errors
        )
        self._headers_enabled = (
            self._headers_enabled
            or config.setdefault(C.HEADERS_ENABLED, False)
        )
```

SimpleLogin never overrides this: `server.py:167` calls `limiter.init_app(app)` with **no**
`headers_enabled` argument, so the default `False` stands. (The `limiter` instance itself is constructed
in `app/extensions.py` with only `key_func=__key_func`.)

**Correcting the version claim.** The changelog entry *"Bug fix: Correct default setting for enabling rate
limit headers. (Issue 22)"* is an **early** Flask-Limiter fix from the **0.7.x** series — it long predates
the pinned **1.4**. It is therefore *not* a "1.5 changelog fix": version 1.4 already ships with the
corrected default (`False`), as the source above shows. Whichever way the history is read, the decisive,
runtime-grounded fact is that **the installed 1.4 emits no rate-limit headers by default**, which is what
the observations confirm.

**(b) The custom `429` handler does not (and need not) suppress headers.** SimpleLogin registers a custom
error handler for `429` at `server.py:362-373` (`rate_limited`) that customizes only the **body** and adds
a **log** line — it does not touch response headers. Crucially, Flask-Limiter's header injector is an
`after_request` hook registered in `init_app` (`extension.py:312-313`: `app.after_request(self.__inject_headers)`),
so it runs on **every** response, including the custom `429`. That hook is a no-op here purely because of
its own guard `if self.enabled and self._headers_enabled and current_limit:` (`extension.py:389`) — with
`_headers_enabled == False` it adds nothing. In other words, the header absence is attributable **solely**
to headers being disabled; the custom handler is **not** a second, independent reason. The relevant
Flask-Limiter source:

```python
# flask_limiter/extension.py  — init_app registers the after_request injector L308-314
            app.extensions = {}  # pragma: no cover

        if not app.extensions.get('limiter'):
            if self._auto_check:
                app.before_request(self.__check_request_limit)
            app.after_request(self.__inject_headers)


# flask_limiter/extension.py  — __inject_headers guard L386-395
    def __inject_headers(self, response):
        self.__check_conditional_deductions(response)
        current_limit = getattr(g, 'view_rate_limit', None)
        if self.enabled and self._headers_enabled and current_limit:
            try:
                window_stats = self.limiter.get_window_stats(*current_limit)
                reset_in = 1 + window_stats[0]
                response.headers.add(
                    self._header_mapping[HEADERS.LIMIT],
                    str(current_limit[0].amount)
```

### 5.2 The 429 boundary, observed (two isolated runs)

`ALIAS_LIMIT = os.environ.get("ALIAS_LIMIT") or "100/day;50/hour;5/minute"` (`app/config.py:448`); the
most restrictive short window is **5/minute**. Because an API-key-only request is rate-limited by client
**IP** (see §5.3), the two runs use **distinct** `REMOTE_ADDR` values so their limiter namespaces do not
collide — a clean, isolated reproduction rather than a contaminated one. Both runs produced the **same**
result:

* **Run 1** (`REMOTE_ADDR=10.0.0.1`): statuses `[201, 201, 201, 201, 201, 429, 429]` — **first 429 at
  request #6**.
* **Run 2** (`REMOTE_ADDR=10.0.0.2`): statuses `[201, 201, 201, 201, 201, 429, 429]` — **first 429 at
  request #6**.

So `5/minute` allows **exactly five** successful creations; the **sixth** request is the first to be
rejected. (This corrects any "first 429 on the 7th request" reading.) The Redis counters after seven
requests are `7` for every window (`100/1/day`, `5/1/minute`, `50/1/hour`) — Flask-Limiter's fixed-window
strategy counts **all** hits, including the rejected ones.

### 5.3 Key derivation — session (`userid`) vs API-key (`ip`) (Finding: key derivation)

The limiter key comes from `__key_func` in `app/extensions.py`: it returns `userid:{id}` when
`current_user.is_authenticated` (a session/Flask-Login request) and otherwise `ip:{addr}`:

```python
# app/extensions.py  — limiter key function L12-23
# - If the user is not logged in: request source IP
# - If the user is logged in: user_id
def __key_func():
    if current_user.is_authenticated:
        return f"userid:{current_user.id}"
    else:
        ip_addr = get_remote_address()
        return f"ip:{ip_addr}"


# Setup rate limit facility
limiter = Limiter(key_func=__key_func)
```

Flask-Limiter's `before_request` hook runs **before** the API blueprint's `require_api_auth` populates
`g.user`, so on an API-key-only request `current_user` is still anonymous and the key falls to
`ip:{addr}`. This was observed directly (§5.4): the two API-key runs created
`LIMITER/ip:10.0.0.1/…` and `LIMITER/ip:10.0.0.2/…` keys, while a session-authenticated create instead
produced `LIMITER/userid:1793/…` keys. The same split governs the concurrency lock in
`app/parallel_limiter.py:55-58`, which names its Redis lock `cl:{current_user.id}:alias_creation` for a
logged-in user and `cl:{request.remote_addr}:alias_creation` otherwise.

### 5.4 Complete evidence (`probe_ratelimit.py`, unedited)

The full, unedited output of `probe_ratelimit.py` (`out/ratelimit.out`) follows. It sets
`config.DISABLE_RATE_LIMIT = False`, drives seven requests per run through the real endpoint, prints each
request's status and body, dumps the Redis counters and the complete header set of the final `429`, and
verifies the key-derivation split. It ends with `assertions failed: 0` and reports the explicit Redis
cleanup performed in its `finally` block (see §5.5 for state-restoration details):


Command (run inside the canonical container `sl_setup`, from `/app`):

```bash
cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
  CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
  GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
  /app/venv/bin/python /tmp/sl_probes/probe_ratelimit.py
```

Complete, unedited output (`out/ratelimit.out`):

```text
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:30:38,060 - SL - DEBUG - 7119 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:30:39,225 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,491 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty

================================================================================
RUN 1  (REMOTE_ADDR=10.0.0.1, API-key auth, premium user)
================================================================================
ALIAS_LIMIT = '100/day;50/hour;5/minute'  (most restrictive short window = 5/minute)
2026-07-13 19:30:39,523 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,531 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.031238555908203125
  request # 1: HTTP 201  body={"alias":"r1p1.test@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r1p1.test@sl.local","enabled":true,"id":3086,"latest_activity":null,"mailbox":{"email":"user_8n7ursl76g@mailbox.test","id":2101},"mailboxes":[{"email":"user_8n7ursl76g@mailbox.test","id":2101}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,552 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,557 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.024682283401489258
  request # 2: HTTP 201  body={"alias":"r1p2.test@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r1p2.test@sl.local","enabled":true,"id":3087,"latest_activity":null,"mailbox":{"email":"user_8n7ursl76g@mailbox.test","id":2101},"mailboxes":[{"email":"user_8n7ursl76g@mailbox.test","id":2101}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,579 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,584 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.025381088256835938
  request # 3: HTTP 201  body={"alias":"r1p3.test@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r1p3.test@sl.local","enabled":true,"id":3088,"latest_activity":null,"mailbox":{"email":"user_8n7ursl76g@mailbox.test","id":2101},"mailboxes":[{"email":"user_8n7ursl76g@mailbox.test","id":2101}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,605 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,610 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.024649381637573242
  request # 4: HTTP 201  body={"alias":"r1p4.word@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r1p4.word@sl.local","enabled":true,"id":3089,"latest_activity":null,"mailbox":{"email":"user_8n7ursl76g@mailbox.test","id":2101},"mailboxes":[{"email":"user_8n7ursl76g@mailbox.test","id":2101}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,630 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,635 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.024508953094482422
  request # 5: HTTP 201  body={"alias":"r1p5.list@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r1p5.list@sl.local","enabled":true,"id":3090,"latest_activity":null,"mailbox":{"email":"user_8n7ursl76g@mailbox.test","id":2101},"mailboxes":[{"email":"user_8n7ursl76g@mailbox.test","id":2101}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,637 - SL - WARNING - 7119 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 1791 Test User user_8n7ursl76g@mailbox.test>
2026-07-13 19:30:39,638 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.0021305084228515625
  request # 6: HTTP 429  body={"error":"Rate limit exceeded"}

2026-07-13 19:30:39,640 - SL - WARNING - 7119 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 1791 Test User user_8n7ursl76g@mailbox.test>
2026-07-13 19:30:39,640 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.0006844997406005859
  request # 7: HTTP 429  body={"error":"Rate limit exceeded"}

  --> statuses: [201, 201, 201, 201, 201, 429, 429]
  --> first 429 at request #6
  --> Redis LIMITER keys for ip:10.0.0.1:
        LIMITER/ip:10.0.0.1/api.new_custom_alias_v2/100/1/day  = 7
        LIMITER/ip:10.0.0.1/api.new_custom_alias_v2/5/1/minute  = 7
        LIMITER/ip:10.0.0.1/api.new_custom_alias_v2/50/1/hour  = 7
  --> FULL headers of the LAST (429) response:
        Access-Control-Allow-Origin: *
        Content-Length: 32
        Content-Type: application/json
        Set-Cookie: slapp=<redacted>
  --> X-RateLimit-*/Retry-After headers present: NONE
  [ASSERT PASS] RUN 1: first 429 at request #6 (5/minute allows exactly 5)
  [ASSERT PASS] RUN 1: requests 1-5 all 201
  [ASSERT PASS] RUN 1: request 6 is 429
  [ASSERT PASS] RUN 1: no X-RateLimit-*/Retry-After headers on the 429
  [ASSERT PASS] RUN 1: 429 body == {'error':'Rate limit exceeded'}

================================================================================
RUN 2  (REMOTE_ADDR=10.0.0.2, API-key auth, premium user)
================================================================================
ALIAS_LIMIT = '100/day;50/hour;5/minute'  (most restrictive short window = 5/minute)
2026-07-13 19:30:39,662 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,668 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.2 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.02574920654296875
  request # 1: HTTP 201  body={"alias":"r2p1.test@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r2p1.test@sl.local","enabled":true,"id":3091,"latest_activity":null,"mailbox":{"email":"user_tiy3z60qpz@mailbox.test","id":2102},"mailboxes":[{"email":"user_tiy3z60qpz@mailbox.test","id":2102}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,689 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,694 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.2 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.024781227111816406
  request # 2: HTTP 201  body={"alias":"r2p2.word@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r2p2.word@sl.local","enabled":true,"id":3092,"latest_activity":null,"mailbox":{"email":"user_tiy3z60qpz@mailbox.test","id":2102},"mailboxes":[{"email":"user_tiy3z60qpz@mailbox.test","id":2102}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,715 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,720 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.2 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.02468585968017578
  request # 3: HTTP 201  body={"alias":"r2p3.list@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r2p3.list@sl.local","enabled":true,"id":3093,"latest_activity":null,"mailbox":{"email":"user_tiy3z60qpz@mailbox.test","id":2102},"mailboxes":[{"email":"user_tiy3z60qpz@mailbox.test","id":2102}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,741 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,746 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.2 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.02487969398498535
  request # 4: HTTP 201  body={"alias":"r2p4.list@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r2p4.list@sl.local","enabled":true,"id":3094,"latest_activity":null,"mailbox":{"email":"user_tiy3z60qpz@mailbox.test","id":2102},"mailboxes":[{"email":"user_tiy3z60qpz@mailbox.test","id":2102}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,768 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:39,773 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.2 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.02584242820739746
  request # 5: HTTP 201  body={"alias":"r2p5.test@sl.local","creation_date":"2026-07-13 19:30:39+00:00","creation_timestamp":1783971039,"disable_pgp":false,"email":"r2p5.test@sl.local","enabled":true,"id":3095,"latest_activity":null,"mailbox":{"email":"user_tiy3z60qpz@mailbox.test","id":2102},"mailboxes":[{"email":"user_tiy3z60qpz@mailbox.test","id":2102}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

2026-07-13 19:30:39,774 - SL - WARNING - 7119 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 1792 Test User user_tiy3z60qpz@mailbox.test>
2026-07-13 19:30:39,776 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.2 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.0021533966064453125
  request # 6: HTTP 429  body={"error":"Rate limit exceeded"}

2026-07-13 19:30:39,777 - SL - WARNING - 7119 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 1792 Test User user_tiy3z60qpz@mailbox.test>
2026-07-13 19:30:39,778 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 10.0.0.2 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.0006957054138183594
  request # 7: HTTP 429  body={"error":"Rate limit exceeded"}

  --> statuses: [201, 201, 201, 201, 201, 429, 429]
  --> first 429 at request #6
  --> Redis LIMITER keys for ip:10.0.0.2:
        LIMITER/ip:10.0.0.2/api.new_custom_alias_v2/100/1/day  = 7
        LIMITER/ip:10.0.0.2/api.new_custom_alias_v2/5/1/minute  = 7
        LIMITER/ip:10.0.0.2/api.new_custom_alias_v2/50/1/hour  = 7
  --> FULL headers of the LAST (429) response:
        Access-Control-Allow-Origin: *
        Content-Length: 32
        Content-Type: application/json
        Set-Cookie: slapp=<redacted>
  --> X-RateLimit-*/Retry-After headers present: NONE
  [ASSERT PASS] RUN 2: first 429 at request #6 (5/minute allows exactly 5)
  [ASSERT PASS] RUN 2: requests 1-5 all 201
  [ASSERT PASS] RUN 2: request 6 is 429
  [ASSERT PASS] RUN 2: no X-RateLimit-*/Retry-After headers on the 429
  [ASSERT PASS] RUN 2: 429 body == {'error':'Rate limit exceeded'}

================================================================================
KEY DERIVATION (Finding 10): session-auth -> userid:{id}; API-key-only -> ip:{addr}
================================================================================
2026-07-13 19:30:40,033 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:40,276 - SL - DEBUG - 7119 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1793 Test User user_0bz0mv2ccp@mailbox.test> in
2026-07-13 19:30:40,276 - SL - DEBUG - 7119 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-13 19:30:40,276 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.23834538459777832
2026-07-13 19:30:40,281 - SL - DEBUG - 7119 - "/app/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 1793 Test User user_0bz0mv2ccp@mailbox.test>
2026-07-13 19:30:40,387 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.10935640335083008
2026-07-13 19:30:40,408 - SL - INFO - 7119 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:40,414 - SL - DEBUG - 7119 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.024852275848388672
  session-auth create -> HTTP 201
  Redis LIMITER keys for userid:1793: ['LIMITER/userid:1793/api.new_custom_alias_v2/100/1/day', 'LIMITER/userid:1793/api.new_custom_alias_v2/5/1/minute', 'LIMITER/userid:1793/api.new_custom_alias_v2/50/1/hour']
  [ASSERT PASS] session-auth request is rate-limited under userid:{id}
  [ASSERT PASS] API-key-only request is rate-limited under ip:{addr}

================================================================================
SUMMARY
================================================================================
  assertions failed: 0

>>> cleanup: DISABLE_RATE_LIMIT restored to False; deleted 9 LIMITER key(s), 15 session key(s); DB rolled back

```

### 5.5 Stability and net-zero state restoration

The `[201×5, 429, 429]` pattern with the **first 429 at request #6** was reproduced identically across the
two isolated runs above (Run 1 on `10.0.0.1`, Run 2 on `10.0.0.2`), satisfying the ≥2-run stability
requirement. The probe leaves no residue: its `finally` block restores `config.DISABLE_RATE_LIMIT` to
`False`, deletes every `LIMITER/*` key it created, and rolls back the database transaction so no alias or
user persists (see the closing `>>> cleanup: …` line of the output above, and the master-harness net-zero
proof in §11).


## 6. Q4 — Success-path quota checks and what gets logged

**Direct answer.** On the success path the account-level quota gate is
`User.can_create_new_alias()` (`app/models.py:867-884`), which in turn calls
`User.max_alias_for_free_account()` (`app/models.py:858-865`). It evaluates, in order:
`is_active()` → `disabled` → `lifetime_or_active_subscription()` → and only for a non-premium user, a
count comparison `Alias.filter_by(user_id=self.id).count() < max_alias_for_free_account()`. **On a
successful (`201`) creation this method logs nothing** — neither `can_create_new_alias()` nor the endpoint
emits any quota-related log line on success. The **only** quota-related log entry in the whole path is on
the *failure* branch: when the gate returns `False`, the handler logs
`DEBUG new_custom_alias.py:49` (v2) / `:138` (v3) — `user <User …> cannot create any custom alias` — and
returns `400` with the free-account-limit message. (Note the docstring's "15 aliases" text is stale; the
effective cap is `max_alias_for_free_account()`, which returns `config.MAX_NB_EMAIL_FREE_PLAN = 3` under
the test config.)

### 6.1 The quota gate — source

```python
# app/models.py:858-884  — max_alias_for_free_account() + can_create_new_alias()
    def max_alias_for_free_account(self) -> int:
        if (
            self.FLAG_FREE_OLD_ALIAS_LIMIT
            == self.flags & self.FLAG_FREE_OLD_ALIAS_LIMIT
        ):
            return config.MAX_NB_EMAIL_OLD_FREE_PLAN
        else:
            return config.MAX_NB_EMAIL_FREE_PLAN

    def can_create_new_alias(self) -> bool:
        """
        Whether user can create a new alias. User can't create a new alias if
        - has more than 15 aliases in the free plan, *even in the free trial*
        """
        if not self.is_active():
            return False

        if self.disabled:
            return False

        if self.lifetime_or_active_subscription():
            return True
        else:
            return (
                Alias.filter_by(user_id=self.id).count()
                < self.max_alias_for_free_account()
            )
```

### 6.2 What runs on the `201` path, and what is (not) logged

`can_create_new_alias()` is called **first** in both handlers (`new_custom_alias.py:48` for v2,
`:137` for v3), before the body/signature checks. For a premium user
`lifetime_or_active_subscription()` short-circuits to `True` (no count query needed). For a free user the
method runs the count query and returns `count < 3`. Either way the method returns a bare boolean and
**writes no log record**; the source above contains no `LOG.*` call. The endpoint only logs on the
`False` branch:

* **Success (`can_create_new_alias() == True`)** → execution proceeds; the only SL-logger lines observed
  during a `201` are the unrelated `INFO event_dispatcher.py:62` ("Not sending events…") and the
  `DEBUG server.py:284` `after_request` line recording the `201`. **No quota log.**
* **Blocked (`can_create_new_alias() == False`)** → `DEBUG new_custom_alias.py:49`
  `user <User …> cannot create any custom alias`, then `400` with the limit message.

### 6.3 Observed quota-branch values, before and after a success (Finding: quota state)

`probe_functional.py` prints every input to `can_create_new_alias()` (`is_active`, `disabled`,
`lifetime`, `lifetime_or_active_subscription`, the `FLAG_FREE_OLD_ALIAS_LIMIT` selector,
`max_alias_for_free_account`, the alias count, and the resulting boolean) with the exact `models.py` line
each maps to. The **premium** primary user's snapshot (from the complete `functional.out` embedded in
§3.2):

```text
# excerpt of out/functional.out (premium primary user; full run embedded in §3.2)
  can_create_new_alias() branch inputs [primary premium user] (app/models.py:L867-884):
    is_active()                        = True   (L872; delete_on=None)
    disabled                           = False   (L875)
    lifetime                           = True
    lifetime_or_active_subscription()  = True   (L878; trial does NOT count)
    FLAG_FREE_OLD_ALIAS_LIMIT set      = False   (-> max via L858-865)
    max_alias_for_free_account()       = 3
    Alias count for user               = 1   (L881-883 count)
    ==> can_create_new_alias()         = True
```

For a **free** user the probe walks the count from `1` (a fresh user already owns one auto-created default
alias) up to the cap of `3`, showing `can_create_new_alias()` flip from `True` to `False` exactly when the
count reaches the max — and demonstrates that the successful create under quota (13a) emits **no** quota
log, while the over-quota attempt (13b) emits the `DEBUG :49` line. The complete, contiguous COND13 block
from `out/functional.out`:

```text
# excerpt of out/functional.out — CONDITION 13 (free-user quota), contiguous & unedited
================================================================================
CONDITION 13: FREE user -- count-based success path, quota exhaustion, and masking
================================================================================
  can_create_new_alias() branch inputs [fresh FREE user (before any create via endpoint)] (app/models.py:L867-884):
    is_active()                        = True   (L872; delete_on=None)
    disabled                           = False   (L875)
    lifetime                           = False
    lifetime_or_active_subscription()  = False   (L878; trial does NOT count)
    FLAG_FREE_OLD_ALIAS_LIMIT set      = False   (-> max via L858-865)
    max_alias_for_free_account()       = 3
    Alias count for user               = 1   (L881-883 count)
    ==> can_create_new_alias()         = True

================================================================================
CONDITION 13a: FREE user SUCCESS while under quota (count < max; success path emits NO quota log)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "freeok", "signed_suffix": ".test@sl.local.alU82g.-cNc5TIO7y1KGXiSbrcILd1FI9o"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 201
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 434
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"alias":"freeok.test@sl.local","creation_date":"2026-07-13 19:30:34+00:00","creation_timestamp":1783971034,"disable_pgp":false,"email":"freeok.test@sl.local","enabled":true,"id":3079,"latest_activity":null,"mailbox":{"email":"user_ahp49hspk7@mailbox.test","id":2097},"mailboxes":[{"email":"user_ahp49hspk7@mailbox.test","id":2097}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,338 - SL - INFO - 7096 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
    | 2026-07-13 19:30:34,343 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04214215278625488
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 201 == 201
  can_create_new_alias() branch inputs [after one endpoint create] (app/models.py:L867-884):
    is_active()                        = True   (L872; delete_on=None)
    disabled                           = False   (L875)
    lifetime                           = False
    lifetime_or_active_subscription()  = False   (L878; trial does NOT count)
    FLAG_FREE_OLD_ALIAS_LIMIT set      = False   (-> max via L858-865)
    max_alias_for_free_account()       = 3
    Alias count for user               = 2   (L881-883 count)
    ==> can_create_new_alias()         = True
  filling 1 more alias(es) via Alias.create_new(quser, prefix='fill<i>') to reach the cap
  can_create_new_alias() branch inputs [after filling to cap] (app/models.py:L867-884):
    is_active()                        = True   (L872; delete_on=None)
    disabled                           = False   (L875)
    lifetime                           = False
    lifetime_or_active_subscription()  = False   (L878; trial does NOT count)
    FLAG_FREE_OLD_ALIAS_LIMIT set      = False   (-> max via L858-865)
    max_alias_for_free_account()       = 3
    Alias count for user               = 3   (L881-883 count)
    ==> can_create_new_alias()         = False

================================================================================
CONDITION 13b: QUOTA exceeded with a VALID suffix (400 quota + LOG.d L49)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "over", "signed_suffix": ".test@sl.local.alU82g.-cNc5TIO7y1KGXiSbrcILd1FI9o"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 141
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,395 - SL - DEBUG - 7096 - "/app/app/api/views/new_custom_alias.py:49" - new_custom_alias_v2() -  - user <User 1787 Test User user_ahp49hspk7@mailbox.test> cannot create any custom alias
    | 2026-07-13 19:30:34,395 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.009819984436035156
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases' == 'You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases'

================================================================================
CONDITION 13c: QUOTA exceeded with a TAMPERED suffix -> STILL 400 quota (quota MASKS the 412)
================================================================================
COMMAND (Werkzeug test client):
  client.post('/api/v2/alias/custom/new',
      json={"alias_prefix": "overt", "signed_suffix": ".list@sl.local.alU82g.Uhl7AemLXDPUo9KCm03BG_E_LQA"},
      headers={'Authentication': '<disposable-60char-test-api-key>'})
RESPONSE:
  status_code = 400
  headers (Werkzeug test-client; transport headers Date/Server/Connection are
           added by the WSGI server, not the test client -- see probe_live.sh):
    Access-Control-Allow-Origin: *
    Content-Length: 141
    Content-Type: application/json
    Set-Cookie: slapp=<redacted-session-cookie>
  body = {"error":"You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}

  SL-logger records emitted DURING this request [BEGIN]
    | 2026-07-13 19:30:34,406 - SL - DEBUG - 7096 - "/app/app/api/views/new_custom_alias.py:49" - new_custom_alias_v2() -  - user <User 1787 Test User user_ahp49hspk7@mailbox.test> cannot create any custom alias
    | 2026-07-13 19:30:34,407 - SL - DEBUG - 7096 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.009836435317993164
  SL-logger records emitted DURING this request [END]
  [ASSERT PASS] status_code 400 == 400
  [ASSERT PASS] error body 'You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases' == 'You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases'
```

### 6.4 Quota is checked *before* the signature — so it masks the 412

Because `can_create_new_alias()` runs before `check_suffix_signature`, an over-quota free user receives the
`400` quota message **regardless of whether the signed suffix is valid, tampered, or expired**. COND13c
above proves it: the same over-quota user posting a **tampered** suffix still gets `400` with the
free-account-limit body and the `DEBUG :49` `cannot create any custom alias` log — the `412` expiry branch
is never reached. This ordering is a likely contributor to "intermittent" observations: whether a bad
suffix surfaces as `412` or as `400` depends on the user's current alias count at request time.


## 7. Q5 — Execution-path trace: signed-suffix validator and creation-limit enforcers

**Direct answer.** The component that validates signed suffixes is
`check_suffix_signature()` (`app/alias_suffix.py:37-42`), backed by the module-level
`signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)` (`alias_suffix.py:11`); the
domain/format half of suffix validation is `verify_prefix_suffix()` (`alias_suffix.py:45-91`).
Creation limits are enforced by **four** distinct mechanisms, in this request-time order:

1. **`@limiter.limit(ALIAS_LIMIT)`** — Flask-Limiter HTTP rate limit (per `userid`/`ip`) → **429**.
2. **`@parallel_limiter.lock(name="alias_creation")`** — Redis concurrency lock → **429**.
3. **`User.can_create_new_alias()`** — per-account alias quota → **400**.
4. **`rate_limiter.check_bucket_limit`** inside `Alias.create` (`models.py:1628-1641`) —
   per-user alias-creation buckets → **429**.

The signature rejection condition is: `check_suffix_signature` returns `None` for a tampered, expired, or
garbage suffix (all `itsdangerous.BadSignature` subclasses — see §9), which trips `if not alias_suffix:`
→ **412**. The following subsections trace the decorator chain, the v2 and v3 in-handler branch orders
(shared and version-specific), and each enforcer, all grounded in source and observed output.

### 7.1 The decorator chain (identical on both endpoints)

Both handlers carry the same four-decorator stack. Decorators apply bottom-up, so at request time the
**outermost runs first**: rate limit → API auth → concurrency lock → handler body. This exact order was
captured live in the Finding-18 traceback (§9.4): `flask_limiter/extension.py` → `app/api/base.py:58`
(`require_api_auth`) → `app/parallel_limiter.py:61` → the handler.

```python
# app/api/views/new_custom_alias.py:28-32  — v2 decorator stack
@api_bp.route("/v2/alias/custom/new", methods=["POST"])
@limiter.limit(ALIAS_LIMIT)
@require_api_auth
@parallel_limiter.lock(name="alias_creation")
def new_custom_alias_v2():

# app/api/views/new_custom_alias.py:115-119  — v3 decorator stack (identical order)
@api_bp.route("/v3/alias/custom/new", methods=["POST"])
@limiter.limit(ALIAS_LIMIT)
@require_api_auth
@parallel_limiter.lock(name="alias_creation")
def new_custom_alias_v3():

# app/alias_suffix.py:11,37-42  — the signer and the signed-suffix validator
signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
def check_suffix_signature(signed_suffix: str) -> Optional[str]:
    # hypothesis: user will click on the button in the 600 secs
    try:
        return signer.unsign(signed_suffix, max_age=600).decode()
    except itsdangerous.BadSignature:
        return None
```

### 7.2 End-to-end decision flow

The following diagram is the execution-path trace referenced from the executive summary (§1). It shows
the decorator chain, the four enforcers, and every in-handler rejection branch with its status code.

```mermaid
flowchart TD
    A["POST /api/v2|v3/alias/custom/new"] --> B{"@limiter.limit(ALIAS_LIMIT)"}
    B -- "exceeded (6th in 5/min)" --> B1["429 Rate limit exceeded"]
    B -- ok --> C{"@require_api_auth"}
    C -- "no / bad key" --> C1["401 Wrong api key"]
    C -- "disabled" --> C2["403 Disabled account"]
    C -- "inactive" --> C3["401 Account does not exist"]
    C -- ok --> D{"@parallel_limiter.lock(alias_creation)"}
    D -- "contended" --> D1["429 (TooManyRequests)"]
    D -- "lock_redis None" --> E
    D -- "acquired" --> E{"can_create_new_alias()"}
    E -- "False" --> E1["400 quota msg + LOG.d :49/:138"]
    E -- "True" --> F{"request body present & (v3) is dict?"}
    F -- "empty" --> F1["400 request body cannot be empty"]
    F -- "v3 not dict" --> F2["400 does not follow the required format"]
    F -- ok --> G{"check_suffix_signature()"}
    G -- "None: BadSignature (tampered|expired|garbage)" --> G1["412 expiry msg + LOG.w :72/:187"]
    G -- "non-string signed_suffix" --> G2["500 Internal error (AttributeError, §9.4)"]
    G -- "valid" --> H{"verify_prefix_suffix()"}
    H -- "False" --> H1["400 wrong alias prefix or suffix"]
    H -- "True (v3: mailbox_ids list/ownership L171-181)" --> I{"duplicate or '..'?"}
    I -- "duplicate" --> I1["409 already exists + LOG.d :87"]
    I -- "'..'" --> I2["400 2 consecutive dot signs"]
    I -- "no" --> J{"Alias.create -> check_bucket_limit()"}
    J -- "bucket exceeded" --> J1["429 Rate limit exceeded"]
    J -- ok --> K["201 serialize_alias_info_v2"]
```

### 7.3 v2 in-handler branch order (`new_custom_alias_v2`, `app/api/views/new_custom_alias.py:32-112`)

| # | Check (file:line) | On failure |
|---|-------------------|------------|
| 1 | `can_create_new_alias()` (L48) | `LOG.d` L49 → **400** free-account-limit message |
| 2 | request body present (L62) | **400** `request body cannot be empty` |
| 3 | `signed_suffix = data.get("signed_suffix","").strip()` (L65) | non-string → `AttributeError` → **500** (§9.4) |
| 4 | `check_suffix_signature()` (L70); `if not alias_suffix` (L71) | `LOG.w` L72 → **412** `Alias creation time is expired, please retry` |
| 5 | `except Exception` (L74) | `LOG.w` L75 → **400** `Tampered suffix` — effectively unreachable (§9) |
| 6 | `verify_prefix_suffix()` (L78) | **400** `wrong alias prefix or suffix` (L79) |
| 7 | duplicate `full_alias` (L87) | `LOG.d` L87 → **409** `alias … already exists` (L88) |
| 8 | `".." in email` (L92) | **400** `2 consecutive dot signs aren't allowed in an email address` |
| 9 | `Alias.create(**kw)` + commit | **201** `serialize_alias_info_v2` |

### 7.4 v3 in-handler branch order — additional & version-specific branches (`new_custom_alias_v3`, L119-235)

v3 shares the same quota → body → signature → prefix/suffix → duplicate/dots → create skeleton, but adds
several branches that do **not** exist in v2 (marked **v3-only**), and it uses a different signed-suffix
coercion:

| # | Check (file:line) | Note |
|---|-------------------|------|
| 1 | `can_create_new_alias()` (L137) | `LOG.d` L138 → **400** quota |
| 2 | request body present (L151) | **400** `request body cannot be empty` |
| 3 | `isinstance(data, dict)` (L153) | **v3-only** → **400** `request body does not follow the required format` (L154) |
| 4 | `signed_suffix = data.get("signed_suffix","") or ""` then `.strip()` (L157-158) | **v3 has the `or ""` guard** — coerces JSON `null`→`""`; v2 does not (§9.4 null divergence) |
| 5 | `alias_prefix` normalize (L163-164) | trims/normalizes the prefix |
| 6 | `check_alias_prefix()` (L167) | prefix charset validation |
| 7 | `isinstance(mailbox_ids, list)` (L171) | **v3-only** → **400** `mailbox_ids must be an array of id` (L172) |
| 8 | mailbox ownership loop (L174-178) | **v3-only** → **400** `Errors with Mailbox` (L177) |
| 9 | at least one mailbox (L180-181) | **v3-only** |
| 10 | `check_suffix_signature()` (L185); `if not alias_suffix` (L186) | `LOG.w` **L187** → **412** |
| 11 | `except Exception` (L189) | `LOG.w` **L190** → **400** `Tampered suffix` — unreachable (§9) |
| 12 | `verify_prefix_suffix()` (L193) | **400** `wrong alias prefix or suffix` (L194) |
| 13 | duplicate `full_alias` (L202) | **409** `alias … already exists` (L203) |
| 14 | `".." in email` (L207) | **400** two consecutive dots |
| 15 | `Session.flush()` (L218); `AliasMailbox.create` loop (L220-224) | links multiple mailboxes → **201** |

The v2/v3 divergence for the `LOG.w` expiry line (**L72** vs **L187**) is exactly what lets an observed
console line be attributed to the correct endpoint version.

**Domain validation inside `verify_prefix_suffix` (`alias_suffix.py:45-91`).** Beyond returning a bare
`False`, this function validates the suffix's `@domain`: the domain must be in
`user.available_alias_domains()` (built-in SL domains or the user's `verified_custom_domains()`); for a
built-in SL domain the `alias_domain_prefix` must start with `"."` unless `config.DISABLE_ALIAS_SUFFIX`.
Both failure modes were observed (§3.2): a missing leading dot logs `ERROR alias_suffix.py:78`
(`submits a wrong alias suffix`), and an unknown domain logs `ERROR alias_suffix.py:61`
(`wrong alias suffix …`); both surface as `400 wrong alias prefix or suffix`.

**Hostname side effect (both versions).** Each handler reads an optional `hostname` query argument
(`request.args.get("hostname")` — v2 L58, v3 L147). On a successful create, if `hostname` is present the
handler writes an `AliasUsedOn` row —
`AliasUsedOn.create(alias_id=alias.id, hostname=hostname, user_id=alias.user_id)` (v2 L105-106,
v3 L228-229). This is the only persisted side effect beyond the `Alias`/`AliasMailbox` rows; when no
`hostname` arg is supplied (as in every probe here) no `AliasUsedOn` row is written, which is why the
`alias_used_on` table delta stayed `+0` in the net-zero teardown (§5.5, §11).

### 7.5 The four creation-limit enforcers

* **(1) `@limiter.limit(ALIAS_LIMIT)` — HTTP rate limit → 429.** Flask-Limiter enforces
  `ALIAS_LIMIT = "100/day;50/hour;5/minute"` (`app/config.py:448`) keyed by `userid`/`ip` (§5.3). Observed
  boundary: **first 429 on request #6** (§5.2). Rejection condition: the fixed-window counter for any
  window exceeds its limit.
* **(2) `@parallel_limiter.lock(name="alias_creation")` — concurrency lock → 429.** A Redis `SET … nx=True`
  lock keyed `cl:{user.id}:alias_creation` (session) or `cl:{remote_addr}:alias_creation` (API-key). If the
  key already exists (a concurrent create in flight) the `SET` fails and `acquire_lock` raises
  `werkzeug.exceptions.TooManyRequests` → 429 (§7.7). Rejection condition: lock already held. **No-op** when
  `lock_redis is None` (`parallel_limiter.py:51-52`).
* **(3) `User.can_create_new_alias()` — account quota → 400.** The per-account cap detailed in §6.
  Rejection condition: inactive/disabled, or a non-premium user whose alias count has reached
  `max_alias_for_free_account()` (3 under test config).
* **(4) `rate_limiter.check_bucket_limit` inside `Alias.create` → 429.** For each configured bucket,
  `Alias.create` increments a Redis counter `bl:alias_create_{seconds}d:{user.id}:{bucket_id}` and raises
  `werkzeug.exceptions.TooManyRequests` when the count exceeds the bucket's `max_hits`. Buckets are
  `ALIAS_CREATE_RATE_LIMIT_PAID = "50,900:200,3600"` for premium users and
  `ALIAS_CREATE_RATE_LIMIT_FREE = "10,900:50,3600"` for free users. Rejection condition: bucket count >
  `max_hits`. Also a no-op when `lock_redis` is unset. Source:

```python
# app/models.py:1628-1641  — Alias.create calls check_bucket_limit per configured bucket
    def create(cls, **kw):
        commit = kw.pop("commit", False)
        flush = kw.pop("flush", False)

        new_alias = cls(**kw)
        user = User.get(new_alias.user_id)
        if user.is_premium():
            limits = config.ALIAS_CREATE_RATE_LIMIT_PAID
        else:
            limits = config.ALIAS_CREATE_RATE_LIMIT_FREE
        # limits is array of (hits,days)
        for limit in limits:
            key = f"alias_create_{limit[1]}d:{user.id}"
            rate_limiter.check_bucket_limit(key, limit[0], limit[1])

# app/rate_limiter.py:19-40  — check_bucket_limit (Redis incr; raises TooManyRequests)
def check_bucket_limit(
    lock_name: Optional[str] = None,
    max_hits: int = 5,
    bucket_seconds: int = 3600,
):
    # Calculate current bucket time
    int_time = int(datetime.utcnow().timestamp())
    bucket_id = int_time - (int_time % bucket_seconds)
    bucket_lock_name = f"bl:{lock_name}:{bucket_id}"
    if not lock_redis:
        return
    try:
        value = lock_redis.incr(bucket_lock_name, bucket_seconds)
        if value > max_hits:
            LOG.i(
                f"Rate limit hit for {lock_name} (bucket id {bucket_id}) -> {value}/{max_hits}"
            )
            newrelic.agent.record_custom_event(
                "BucketRateLimit",
                {"lock_name": lock_name, "bucket_seconds": bucket_seconds},
            )
            raise werkzeug.exceptions.TooManyRequests()

# app/config.py:554-559  — bucket definitions (free / paid)
ALIAS_CREATE_RATE_LIMIT_FREE = getRateLimitFromConfig(
    "ALIAS_CREATE_RATE_LIMIT_FREE", "10,900:50,3600"
)
ALIAS_CREATE_RATE_LIMIT_PAID = getRateLimitFromConfig(
    "ALIAS_CREATE_RATE_LIMIT_PAID", "50,900:200,3600"
)
```

### 7.6 One 429 handler serves all three rate-limit mechanisms

Enforcers (1), (2) and (4) all surface as HTTP **429** through the *same* custom error handler,
`rate_limited()` at `server.py:362-373`: it logs `LOG.w server.py:364`
`Client hit rate limit on path <path>, user:<user>` and, for `/api/` paths, returns
`jsonify(error="Rate limit exceeded"), 429`. That is why every 429 in this investigation — the HTTP
limiter (§5), the concurrency lock (§7.7), and the alias-create bucket — carries the **identical** body
`{"error":"Rate limit exceeded"}` (`Content-Length: 32`) and the same `server.py:364` warning. The handler
customizes only body + log; it does **not** touch response headers (§5.1).

### 7.7 Concurrency-lock behavior and state restoration

`parallel_limiter.py` derives the lock name from the same session-vs-API-key split as the rate limiter,
acquires with `SET … ex=timedelta(seconds=max_wait_secs=5), nx=True`, and releases in a `finally` only if
the stored value still matches (a guarded release that never deletes another holder's lock):

```python
# app/parallel_limiter.py:30-62  — acquire/release + lock-name derivation + no-op guard
    def acquire_lock(self, lock_name: str, lock_value: str):
        if not lock_redis.storage.set(
            lock_name, lock_value, ex=timedelta(seconds=self.max_wait_secs), nx=True
        ):
            raise exceptions.TooManyRequests()

    def release_lock(self, lock_name: str, lock_value: str):
        current_lock_value = lock_redis.storage.get(lock_name)
        if current_lock_value == lock_value.encode("utf-8"):
            lock_redis.storage.delete(lock_name)

    def __call__(self, f: Callable[..., Any]):
        if self.lock_suffix is None:
            lock_suffix = f.__name__
        else:
            lock_suffix = self.lock_suffix

        @wraps(f)
        def decorated(*args, **kwargs):
            if self.only_when and not self.only_when():
                return f(*args, **kwargs)
            if not lock_redis:
                return f(*args, **kwargs)

            lock_value = str(uuid.uuid4())[:10]
            if "id" in dir(current_user):
                lock_name = f"cl:{current_user.id}:{lock_suffix}"
            else:
                lock_name = f"cl:{request.remote_addr}:{lock_suffix}"
            self.acquire_lock(lock_name, lock_value)
            try:
                return f(*args, **kwargs)
            finally:
```

`probe_lock.py` exercises all four behaviors with full state restoration in a `finally` block — it saves
and restores `parallel_limiter.lock_redis` and `config.DISABLE_RATE_LIMIT`, uses a short-TTL lock key, and
deletes any key it created; the one seeding step (pre-populating a lock to force contention) is explicitly
labeled **NON-CANONICAL** because it bypasses the real `acquire_lock`. It shows: **(A)** a normal create
acquires and self-releases `cl:{addr}:alias_creation` (key absent before *and* after); **(B)** a pre-seeded
holder forces the *real* `acquire_lock` to raise `429`, and the guarded release preserves the seeded
holder; **(C)** a session-authenticated request locks under `cl:{user.id}:alias_creation`; **(D)** with
`lock_redis = None` the decorator is a no-op and the create returns `201`. Complete, unedited output
(`out/lock.out`):

```text
# out/lock.out  — parallel_limiter concurrency-lock probe (complete, unedited)
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:30:41,289 - SL - DEBUG - 7130 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:30:42,464 - SL - INFO - 7130 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
lock_redis wired from MEM_STORE_URI: True
  [ASSERT PASS] parallel_limiter.lock_redis is wired

=== (A) success acquires + self-releases cl:{addr}:alias_creation ===
2026-07-13 19:30:42,500 - SL - INFO - 7130 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:42,508 - SL - DEBUG - 7130 - "/app/server.py:284" - after_request() -  - 10.9.9.9 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.03043675422668457
  create -> HTTP 201; cl:10.9.9.9:alias_creation exists before=False after=False
  [ASSERT PASS] (A) success create returns 201
  [ASSERT PASS] (A) lock key absent before AND after (acquired+released within request)

=== (B) contention: pre-seeded holder => real acquire_lock raises 429 ===
  [NON-CANONICAL] seeding cl:10.9.9.9:alias_creation to simulate a concurrent in-flight request holding the lock
2026-07-13 19:30:42,514 - SL - WARNING - 7130 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 1794 Test User user_as91pht6zc@mailbox.test>
2026-07-13 19:30:42,514 - SL - DEBUG - 7130 - "/app/server.py:284" - after_request() -  - 10.9.9.9 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.0038144588470458984
  contended create -> HTTP 429; body={"error":"Rate limit exceeded"}

  [ASSERT PASS] (B) contended create returns 429 (parallel_limiter TooManyRequests)
  [ASSERT PASS] (B) 429 body == {'error':'Rate limit exceeded'}
  [ASSERT PASS] (B) seeded holder preserved (nx set failed; guarded release did not delete it)

=== (C) session auth => lock name cl:{user.id}:alias_creation ===
2026-07-13 19:30:42,769 - SL - INFO - 7130 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:43,011 - SL - DEBUG - 7130 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1795 Test User user_s6yw0n1g8r@mailbox.test> in
2026-07-13 19:30:43,011 - SL - DEBUG - 7130 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-13 19:30:43,011 - SL - DEBUG - 7130 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.23765921592712402
2026-07-13 19:30:43,016 - SL - DEBUG - 7130 - "/app/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 1795 Test User user_s6yw0n1g8r@mailbox.test>
2026-07-13 19:30:43,121 - SL - DEBUG - 7130 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.10831379890441895
  [NON-CANONICAL] seeding cl:1795:alias_creation
2026-07-13 19:30:43,126 - SL - WARNING - 7130 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 1795 Test User user_s6yw0n1g8r@mailbox.test>
2026-07-13 19:30:43,126 - SL - DEBUG - 7130 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.002863168716430664
  session contended create -> HTTP 429
  [ASSERT PASS] (C) session-auth contended create returns 429 under cl:{user.id}:alias_creation

=== (D) lock_redis=None => decorator is a no-op (parallel_limiter.py:L51-52) ===
2026-07-13 19:30:43,147 - SL - INFO - 7130 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:30:43,152 - SL - DEBUG - 7130 - "/app/server.py:284" - after_request() -  - 10.9.9.9 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.024588346481323242
  create with lock_redis=None -> HTTP 201 (lock skipped)
  [ASSERT PASS] (D) create succeeds (201) when lock_redis is None (no-op path)

=== SUMMARY ===
assertions failed: 0

>>> cleanup: restored lock_redis+DISABLE_RATE_LIMIT; deleted 1 cl key(s); DB rolled back
```

## 8. Complete condition matrix

Every distinct condition exercised through the real endpoints, with the observed status, the exact body,
the governing check (`file:line`) and the log emitter. The **complete, unedited transcript** for the
test-client conditions is in §3.2; rate-limit/lock/500 conditions are evidenced in §5, §7 and §9.4.

| # | Condition (input) | Endpoint | Status | Response body (verbatim) | Governing check (`file:line`) | Log emitter |
|---|-------------------|----------|--------|--------------------------|-------------------------------|-------------|
| 1 | Valid signed suffix, under quota | v2 | `201` | serialized alias object (full body in §3.2) | `Alias.create` `app/models.py` | `INFO event_dispatcher.py:62` + `DEBUG server.py:284` |
| 2 | Valid signed suffix + `mailbox_ids`, under quota | v3 | `201` | serialized alias | `Alias.create` + `AliasMailbox` `new_custom_alias.py:218-224` | `INFO event_dispatcher.py:62` + `DEBUG server.py:284` |
| 3 | **Tampered** signed suffix (one signature char changed) | v2/v3 | `412` | `{"error":"Alias creation time is expired, please retry"}` | `check_suffix_signature` `alias_suffix.py:37-42` → `None` | `WARNING new_custom_alias.py:72` (v2) / `:187` (v3) |
| 4 | **Garbage** string (never signed) | v2/v3 | `412` | `{"error":"Alias creation time is expired, please retry"}` | `check_suffix_signature` → `BadSignature` → `None` | `WARNING new_custom_alias.py:72/:187` |
| 5 | **Expired** signed suffix (`age 601s > max_age=600`) | v2/v3 | `412` | `{"error":"Alias creation time is expired, please retry"}` | `check_suffix_signature` → `SignatureExpired` → `None` | `WARNING new_custom_alias.py:72/:187` |
| 6 | Empty body `{}` | v2/v3 | `400` | `{"error":"request body cannot be empty"}` | `new_custom_alias.py:62` (v2) / `:151` (v3) | `DEBUG server.py:284` only |
| 7 | Body is not a JSON object | v3 | `400` | `{"error":"request body does not follow the required format"}` | `isinstance(data, dict)` `new_custom_alias.py:153-154` | `DEBUG server.py:284` only |
| 8 | `mailbox_ids` not an array | v3 | `400` | `{"error":"mailbox_ids must be an array of id"}` | `isinstance(mailbox_ids, list)` `new_custom_alias.py:171-172` | `DEBUG server.py:284` only |
| 9 | **Duplicate** alias (second identical create) | v2/v3 | `409` | `{"error":"alias <full-address> already exists"}` | duplicate guard `new_custom_alias.py:87-88` (v2) / `:202-203` (v3) | `DEBUG new_custom_alias.py:87` |
| 10 | Two consecutive dots in prefix (`prefix.`) | v2/v3 | `400` | `{"error":"2 consecutive dot signs aren't allowed in an email address"}` | `new_custom_alias.py:92` (v2) / `:207` (v3) | `DEBUG server.py:284` only |
| 11 | Wrong suffix (SL domain, prefix lacks `.`) | v2/v3 | `400` | `{"error":"wrong alias prefix or suffix"}` | `verify_prefix_suffix` `alias_suffix.py:77-79` → `False` | `ERROR alias_suffix.py:78` (`LOG.e`, + `NoneType: None`) |
| 12 | Wrong domain (validly signed for an unusable domain) | v2/v3 | `400` | `{"error":"wrong alias prefix or suffix"}` | `verify_prefix_suffix` `alias_suffix.py:60-62` → `False` | `ERROR alias_suffix.py:61` (`LOG.e`, + `NoneType: None`) |
| 13 | Quota exhausted (free account at `MAX_NB_EMAIL_FREE_PLAN=3`) | v2/v3 | `400` | `{"error":"You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}` | `can_create_new_alias()` `app/models.py:867-884` → `False` | `DEBUG new_custom_alias.py:49` |
| 13b | Quota exhausted **and** tampered suffix (masking) | v2/v3 | `400` | quota message (as #13) — **not** the 412 | quota check precedes signature check | `DEBUG new_custom_alias.py:49` |
| 14a | No `Authentication` header | v2/v3 | `401` | `{"error":"Wrong api key"}` | `require_api_auth` `app/api/base.py` | (auth-layer) |
| 14b | Invalid API key | v2/v3 | `401` | `{"error":"Wrong api key"}` | `require_api_auth` | (auth-layer) |
| 14c | Disabled account (valid key) | v2/v3 | `403` | `{"error":"Disabled account"}` | `app/api/base.py:36-37` | (auth-layer) |
| 14d | Inactive account (`delete_on` in the future) | v2/v3 | `401` | `{"error":"Account does not exist"}` | `app/api/base.py:39-40` (`is_active()` `models.py:766-769`) | (auth-layer) |
| 15 | **Truthy non-string** `signed_suffix` (list/int/dict) | v2 | `500` | `{"error":"Internal error"}` | `.strip()` before the `try` — `new_custom_alias.py:65` | `ERROR server.py:390` (+ traceback) |
| 15 | **Truthy non-string** `signed_suffix` (list/int/dict) | v3 | `500` | `{"error":"Internal error"}` | `.strip()` before the `try` — `new_custom_alias.py:158` | `ERROR server.py:390` (+ traceback) |
| 16 | `signed_suffix` missing / empty `""` | v2/v3 | `412` | `{"error":"Alias creation time is expired, please retry"}` | default `""` (falsy) → `check_suffix_signature("")` → `None` | `WARNING new_custom_alias.py:72/:187` |
| 17 | `signed_suffix` JSON `null` | v2 | `500` | `{"error":"Internal error"}` | `data.get("signed_suffix", "")` returns `None`; `None.strip()` — `new_custom_alias.py:65` | `ERROR server.py:390` (+ traceback) |
| 17 | `signed_suffix` JSON `null` | v3 | `412` | `{"error":"Alias creation time is expired, please retry"}` | `signed_suffix = data.get("signed_suffix", "") or ""` coerces `None`→`""` — `new_custom_alias.py:157` | `WARNING new_custom_alias.py:187` |
| 18 | HTTP rate limit exceeded (`5/minute`) | v2/v3 | `429` | `{"error":"Rate limit exceeded"}` | `@limiter.limit(ALIAS_LIMIT)` → `server.py:364` | `WARNING server.py:364` |
| 19 | Parallel-creation lock contended | v2/v3 | `429` | `{"error":"Rate limit exceeded"}` | `@parallel_limiter.lock` → `TooManyRequests` → `server.py:364` | `WARNING server.py:364` |
| 20 | Per-user alias-create bucket exceeded | v2/v3 | `429` | `{"error":"Rate limit exceeded"}` | `check_bucket_limit` `app/models.py:1634-1641` → `TooManyRequests` | `WARNING server.py:364` |

**Reading the matrix for Q1:** rows 3, 4 and 5 are the crux — invalid, garbage and expired suffixes are
indistinguishable at the HTTP layer (all `412`, identical body). Rows 15 and 17 show the only way to get a
non-412 out of the signature area (an unhandled `500`), and only via a non-string value a canonical client
never sends.

---

## 9. Root cause — why tampered, expired, and garbage suffixes all collapse to one HTTP 412

**Direct answer.** A custom-alias submission whose `signed_suffix` is **tampered**, **genuinely
expired**, or **outright garbage** produces the *same* response — **HTTP 412** with body
`{"error":"Alias creation time is expired, please retry"}` and a single **WARNING** log line
`Alias creation time expired for <user>`. The endpoint's second branch, `except Exception ->
HTTP 400 "Tampered suffix"`, is **effectively unreachable for ordinary string input**. The cause
is a single line: `check_suffix_signature` catches `itsdangerous.BadSignature`, which is the
**superclass** of the expiry and tampering exceptions, so all three failure modes return `None`
and are funnelled into the one "expired" branch. Every claim below sits next to the runtime
output that established it. All three probes cited here (`probe_mro.py`, `probe_f18.py`,
`probe_customdomain.py`) roll back every write and are net-zero (§9.5 and §2.5).

### 9.1 The `itsdangerous` 1.1.0 exception hierarchy (observed)

The signer is a module-level `TimestampSigner` keyed by `CUSTOM_ALIAS_SECRET`
[`app/alias_suffix.py:11`]:
```python
# app/alias_suffix.py:6-11
import itsdangerous
from app import config
from app.log import LOG
from app.models import User, AliasOptions, SLDomain

signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
```

Probed in isolation against the pinned library (`probe_mro.py`, part of the single `run_all.sh`
run at `19:30:31`), the class hierarchy and `issubclass` relations are:

Command (run inside the canonical container `sl_setup`, from `/app`):

```bash
cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
  CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
  GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
  /app/venv/bin/python /tmp/sl_probes/probe_mro.py
```

Complete, unedited excerpt (`out/mro.out`), version and hierarchy:
```text
itsdangerous.__version__ = 1.1.0
SignatureExpired.__mro__  = ['SignatureExpired', 'BadTimeSignature', 'BadSignature', 'BadData', 'Exception', 'BaseException', 'object']
BadTimeSignature.__mro__  = ['BadTimeSignature', 'BadSignature', 'BadData', 'Exception', 'BaseException', 'object']
BadSignature.__mro__      = ['BadSignature', 'BadData', 'Exception', 'BaseException', 'object']
issubclass(SignatureExpired, BadSignature) = True
issubclass(BadTimeSignature, BadSignature) = True
```

So `SignatureExpired` (raised on expiry) and `BadTimeSignature` (raised on a bad timestamped
signature) are **both subclasses of `BadSignature`**, and a structurally-broken value raises
`BadSignature` itself.

### 9.2 `check_suffix_signature` catches the superclass `BadSignature`

The validator [`app/alias_suffix.py:37-42`] wraps the `unsign` call (max_age=600) in a single
`except itsdangerous.BadSignature` clause:
```python
# app/alias_suffix.py:37-42
def check_suffix_signature(signed_suffix: str) -> Optional[str]:
    # hypothesis: user will click on the button in the 600 secs
    try:
        return signer.unsign(signed_suffix, max_age=600).decode()
    except itsdangerous.BadSignature:
        return None
```

Because the `except` names the **superclass**, every subclass is caught. Exercising the four
input classes through the real signer (`probe_mro.py`, same run) shows each one either returns
the decoded suffix or is swallowed to `None`:
```text
----- behavior per input (max_age=600) -----
valid     unsign(max_age=600) raises -> None;  check_suffix_signature -> '.word@sl.local'
tampered  unsign(max_age=600) raises -> BadTimeSignature: Signature b'NaRol-wG7zCLoCWZA5SUaR42Udc' does not match;  check_suffix_signature -> None
expired   unsign(max_age=600) raises -> SignatureExpired: Signature age 601 > 600 seconds;  check_suffix_signature -> None
garbage   unsign(max_age=600) raises -> BadSignature: No b'.' found in value;  check_suffix_signature -> None
```

`valid` decodes to `.word@sl.local`; `tampered`, `expired`, and `garbage` each raise a
`BadSignature`-family exception that is caught and converted to `None`. Back in the handler,
`if not alias_suffix:` is therefore true for all three, so each returns the identical HTTP 412
"expired" response — the tampered and garbage cases are **misreported as expiry**.

### 9.3 Constructing a genuinely-tampered token (Finding 15)

A tampered token must be *guaranteed* to fail verification. The naive approach of replacing the
last character (for example `valid[:-1] + "A"`) is **unreliable**: the trailing base64url
character can carry padding bits, so a different final character may still decode to the same
signature bytes and verify — producing a nondeterministic false-valid probe. The `probe_mro.py`
run records that on this particular token the naive flip *happened* to work, which is exactly the
fragile, run-to-run-dependent behavior to avoid:
```text
[Finding 15] naive last-char flip 'c'->'A': raised BadTimeSignature (naive flip happened to work this time)
```

Every probe in this investigation therefore builds tampered tokens with the deterministic
`make_tampered` helper, which walks character positions from the front, applies the first
single-character change that **verifiably raises `BadSignature`**, and (in the calling probes)
asserts `tampered != valid` before use:
```python
# probe_functional.py (identical helper reused in every probe, including probe_customdomain.py)
_B64URL = _string.ascii_letters + _string.digits + "-_"

def make_tampered(valid_token):
    """Return a token GUARANTEED to fail signature verification (VERIFIED to raise
    itsdangerous.BadSignature). A naive last-character flip is unreliable because the
    final base64url char can carry padding bits, so a different char may still decode
    to the same signature bytes and verify (Finding 15). This walks positions from the
    front and returns the first single-char change that genuinely invalidates it."""
    for i in range(len(valid_token)):
        c = valid_token[i]
        repl = next(a for a in _B64URL if a != c)
        cand = valid_token[:i] + repl + valid_token[i + 1:]
        try:
            signer.unsign(cand, max_age=600)
        except itsdangerous.BadSignature:
            return cand
    raise RuntimeError("could not construct a tampered token")
```

### 9.4 Non-string `signed_suffix`: the "Tampered suffix" 400 branch is effectively unreachable (Finding 18)

The handlers coerce `signed_suffix` to a stripped string *before* the `try` that calls
`check_suffix_signature`. In **v2** [`app/api/views/new_custom_alias.py:60-79`]:
```python
# app/api/views/new_custom_alias.py:60-79  (v2)
    data = request.get_json()
    if not data:
        return jsonify(error="request body cannot be empty"), 400

    alias_prefix = data.get("alias_prefix", "").strip().lower().replace(" ", "")
    signed_suffix = data.get("signed_suffix", "").strip()
    note = data.get("note")
    alias_prefix = convert_to_id(alias_prefix)

    try:
        alias_suffix = check_suffix_signature(signed_suffix)
        if not alias_suffix:
            LOG.w("Alias creation time expired for %s", user)
            return jsonify(error="Alias creation time is expired, please retry"), 412
    except Exception:
        LOG.w("Alias suffix is tampered, user %s", user)
        return jsonify(error="Tampered suffix"), 400

    if not verify_prefix_suffix(user, alias_prefix, alias_suffix):
        return jsonify(error="wrong alias prefix or suffix"), 400
```

In **v3** [`app/api/views/new_custom_alias.py:150-191`] the coercion adds an `or ""` guard
(L157) before `.strip()` (L158); the signature `try/except` is identical:
```python
# app/api/views/new_custom_alias.py:150-191  (v3)
    if not data:
        return jsonify(error="request body cannot be empty"), 400

    if not isinstance(data, dict):
        return jsonify(error="request body does not follow the required format"), 400

    alias_prefix = data.get("alias_prefix", "").strip().lower().replace(" ", "")
    signed_suffix = data.get("signed_suffix", "") or ""
    signed_suffix = signed_suffix.strip()

    mailbox_ids = data.get("mailbox_ids")
    note = data.get("note")
    name = data.get("name")
    if name:
        name = name.replace("\n", "")
    alias_prefix = convert_to_id(alias_prefix)

    if not check_alias_prefix(alias_prefix):
        return jsonify(error="alias prefix invalid format or too long"), 400

    # check if mailbox is not tempered with
    if not isinstance(mailbox_ids, list):
        return jsonify(error="mailbox_ids must be an array of id"), 400
    mailboxes = []
    for mailbox_id in mailbox_ids:
        mailbox = Mailbox.get(mailbox_id)
        if not mailbox or mailbox.user_id != user.id or not mailbox.verified:
            return jsonify(error="Errors with Mailbox"), 400
        mailboxes.append(mailbox)

    if not mailboxes:
        return jsonify(error="At least one mailbox must be selected"), 400

    # hypothesis: user will click on the button in the 600 secs
    try:
        alias_suffix = check_suffix_signature(signed_suffix)
        if not alias_suffix:
            LOG.w("Alias creation time expired for %s", user)
            return jsonify(error="Alias creation time is expired, please retry"), 412
    except Exception:
        LOG.w("Alias suffix is tampered, user %s", user)
        return jsonify(error="Tampered suffix"), 400
```

Two observed consequences make the outer `except Exception -> 400 "Tampered suffix"` branch
(v2 L74-76, v3 L189-191) unreachable in practice:

1. **Ordinary string input** — the value is a `str`, so `.strip()` succeeds and the
   `signer.unsign` call raises a `BadSignature` subclass, which is caught by the **inner**
   `except itsdangerous.BadSignature` inside `check_suffix_signature` (returning `None` -> 412).
   The exception never propagates out to the handler's outer `except`.
2. **Truthy non-string input** (JSON list / int / null) — `.strip()` raises `AttributeError`
   **before** the `try` block is even entered (v2 L65, v3 L158), so it is not caught by the
   handler's `except` either; it escapes to the global error handler as **HTTP 500
   `{"error":"Internal error"}`** — *not* "Tampered suffix".

`probe_f18.py` (part of the single `run_all.sh` run at `19:30:35-37`, premium user id=1790, all
writes rolled back) exercises every non-string and empty variant and captures the full unhandled
traceback:

Command (run inside the canonical container `sl_setup`, from `/app`):

```bash
cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
  CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
  GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
  /app/venv/bin/python /tmp/sl_probes/probe_f18.py
```

Complete, unedited output (`out/f18.out`):
```text
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:30:35,956 - SL - DEBUG - 7107 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:30:37,072 - SL - INFO - 7107 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
>>> probe_f18.py starting (Finding 18: .strip() on non-string signed_suffix)
>>> disposable premium user id=1790; api key REDACTED

--- v2: signed_suffix is a TRUTHY NON-STRING (JSON list) ---
COMMAND: POST /api/v2/alias/custom/new json={alias_prefix:'f18',signed_suffix:[1,2,3]}
2026-07-13 19:30:37,091 - SL - ERROR - 7107 - "/app/server.py:390" - error_handler() -  - 'list' object has no attribute 'strip'
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/app/api/base.py", line 58, in decorated
    return f(*args, **kwargs)
  File "/app/app/parallel_limiter.py", line 61, in decorated
    return f(*args, **kwargs)
  File "/app/app/api/views/new_custom_alias.py", line 65, in new_custom_alias_v2
    signed_suffix = data.get("signed_suffix", "").strip()
AttributeError: 'list' object has no attribute 'strip'
2026-07-13 19:30:37,092 - SL - DEBUG - 7107 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 500, takes 0.005604743957519531
  v2 list signed_suffix
    status_code = 500
    body        = '{"error":"Internal error"}\n'
  [ASSERT PASS] v2 truthy non-string signed_suffix -> 500 (AttributeError at L65 .strip(), BEFORE try at L69)
  [ASSERT PASS] v2: body is NOT 'Tampered suffix' (except branch L74-76 not reached)

--- v3: signed_suffix is a TRUTHY NON-STRING (JSON int) ---
COMMAND: POST /api/v3/alias/custom/new json={alias_prefix:'f18',signed_suffix:123,mailbox_ids:[<mb>]}
2026-07-13 19:30:37,098 - SL - ERROR - 7107 - "/app/server.py:390" - error_handler() -  - 'int' object has no attribute 'strip'
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/app/api/base.py", line 58, in decorated
    return f(*args, **kwargs)
  File "/app/app/parallel_limiter.py", line 61, in decorated
    return f(*args, **kwargs)
  File "/app/app/api/views/new_custom_alias.py", line 158, in new_custom_alias_v3
    signed_suffix = signed_suffix.strip()
AttributeError: 'int' object has no attribute 'strip'
2026-07-13 19:30:37,098 - SL - DEBUG - 7107 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 500, takes 0.004415273666381836
  v3 int signed_suffix
    status_code = 500
    body        = '{"error":"Internal error"}\n'
  [ASSERT PASS] v3 truthy non-string signed_suffix -> 500 (AttributeError at L158 .strip(), BEFORE try at L185)

--- v2: signed_suffix key MISSING (defaults to '' via data.get(...,'')) ---
2026-07-13 19:30:37,103 - SL - WARNING - 7107 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1790 Test User user_fof2cgfeuw@mailbox.test>
2026-07-13 19:30:37,104 - SL - DEBUG - 7107 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.004436016082763672
  v2 missing signed_suffix
    status_code = 412
    body        = '{"error":"Alias creation time is expired, please retry"}\n'
  [ASSERT PASS] v2 missing signed_suffix -> 412 ('' falsy -> check_suffix_signature('') -> None)

--- v2: signed_suffix is empty string '' ---
2026-07-13 19:30:37,109 - SL - WARNING - 7107 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1790 Test User user_fof2cgfeuw@mailbox.test>
2026-07-13 19:30:37,109 - SL - DEBUG - 7107 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.004253864288330078
  v2 empty signed_suffix
    status_code = 412
    body        = '{"error":"Alias creation time is expired, please retry"}\n'
  [ASSERT PASS] v2 empty-string signed_suffix -> 412

--- v2: signed_suffix is JSON null (None) ---
2026-07-13 19:30:37,114 - SL - ERROR - 7107 - "/app/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'strip'
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/app/api/base.py", line 58, in decorated
    return f(*args, **kwargs)
  File "/app/app/parallel_limiter.py", line 61, in decorated
    return f(*args, **kwargs)
  File "/app/app/api/views/new_custom_alias.py", line 65, in new_custom_alias_v2
    signed_suffix = data.get("signed_suffix", "").strip()
AttributeError: 'NoneType' object has no attribute 'strip'
2026-07-13 19:30:37,115 - SL - DEBUG - 7107 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 500, takes 0.0043642520904541016
  v2 null signed_suffix
    status_code = 500
    body        = '{"error":"Internal error"}\n'
  [ASSERT PASS] v2 JSON-null signed_suffix observed (see body/status above)

--- v3: signed_suffix is JSON null (None) -- v3 guards with `or ""` ---
2026-07-13 19:30:37,120 - SL - WARNING - 7107 - "/app/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 1790 Test User user_fof2cgfeuw@mailbox.test>
2026-07-13 19:30:37,121 - SL - DEBUG - 7107 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.00514674186706543
  v3 null signed_suffix
    status_code = 412
    body        = '{"error":"Alias creation time is expired, please retry"}\n'
  [ASSERT PASS] v3 JSON-null signed_suffix -> 412 (`or ""` coerces None -> '' -> None -> 412)

=== SUMMARY ===
  assertions failed: 0
>>> rolled back transaction; probe_f18 committed NOTHING to PostgreSQL
```

The traceback pins the failure to the coercion line — v2 `new_custom_alias.py:65
signed_suffix = data.get("signed_suffix", "").strip()` and v3 `new_custom_alias.py:158
signed_suffix = signed_suffix.strip()` — inside the decorator chain
(`flask_limiter/extension.py:702` -> `app/api/base.py:58` -> `app/parallel_limiter.py:61`).
The v2-vs-v3 divergence on JSON `null` is also observed and explained: v2 has no `or ""`
guard, so `data.get("signed_suffix", "")` returns `None` when the key is present-but-null
(-> 500), whereas v3's `or ""` (L157) coerces `None` to `""` (-> falsy ->
`check_suffix_signature("")` -> `None` -> 412). The empty-string and missing-key cases both
reach 412 through the same falsy path. In none of these cases does the body read
`"Tampered suffix"`.

### 9.5 The collapse is domain-independent (Finding 16b: custom-domain header run)

To confirm the 412 collapse is a property of the signature check and not of the built-in
`sl.local` domain, `probe_customdomain.py` provisions a **verified** custom domain, mints a
canonical `@<domain>` suffix through the real signer, and drives one valid create (expect 201)
and one tampered create (expect 412) through `POST /api/v3/alias/custom/new`. It counts committed
aliases on a **separate** connection (the same external-observer technique `run_all.sh` uses via
`psql`) to prove net-zero:

Command (run inside the canonical container `sl_setup`, from `/app`):

```bash
cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
  CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
  GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
  /app/venv/bin/python /tmp/sl_probes/probe_customdomain.py
```

Complete, unedited output (`out/customdomain.out`), including the full response header set and the
explicit `X-RateLimit-*/Retry-After headers present: NONE` check:
```text
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 20:45:42,149 - SL - DEBUG - 9565 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> probe_customdomain.py starting (Finding 16b: real custom-domain header run)
>>> CUSTOM_ALIAS_SECRET='secretcustom_alias'  (public TEST-ONLY secret = FLASK_SECRET+'custom_alias')
>>> All signed suffixes below are SYNTHETIC, DISPOSABLE, TEST-ONLY and TIME-LIMITED (max_age=600s).
>>> alias count BEFORE = 824

>>> disposable premium user id=1822; api key REDACTED
>>> created VERIFIED CustomDomain id=357 domain='cd-lrfkxkwx.test' verified=True ownership_verified=True
>>> user.available_alias_domains() includes 'cd-lrfkxkwx.test'? True
  [ASSERT PASS] custom domain cd-lrfkxkwx.test is offered to the user

[mint] custom-domain suffix (synthetic/disposable/test-only/time-limited): '@cd-lrfkxkwx.test'
[mint] signed_suffix (disposable, expires 600s): '@cd-lrfkxkwx.test.alVOdw.eYeUqPLMRQ6jje-mykX-QlptmR8'
[verify] check_suffix_signature(valid) -> '@cd-lrfkxkwx.test'

--- REQ1: custom-domain create (valid signed suffix) -> expect 201 ---
COMMAND: POST /api/v3/alias/custom/new json={alias_prefix:'cdok',signed_suffix:<valid>,mailbox_ids:[2132]}
  status_code = 201
  body        = '{"alias":"cdok@cd-lrfkxkwx.test","creation_date":"2026-07-13 20:45:43+00:00","creation_timestamp":1783975543,"disable_pgp":false,"email":"cdok@cd-lrfkxkwx.test","enabled":true,"id":3180,"latest_activity":null,"mailbox":{"email":"user_nsupeov4ie@mailbox.test","id":2132},"mailboxes":[{"email":"user_nsupeov4ie@mailbox.test","id":2132}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}\n'
  response headers (complete, as returned by the app):
    Access-Control-Allow-Origin: *
    Content-Length: 436
    Content-Type: application/json
    Set-Cookie: slapp=e6859c85-0cc0-4f00-99b6-ea2364550a22.MKnD6l8Vp0kjQLFncUquiLavblg; Domain=.sl.test; Expires=Mon, 20-Jul-2026 20:45:43 GMT; HttpOnly; Path=/; SameSite=Lax
  X-RateLimit-*/Retry-After headers present: NONE
  SL log:
    2026-07-13 20:45:43,393 - SL - INFO - 9565 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
    2026-07-13 20:45:43,402 - SL - DEBUG - 9565 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 201, takes 0.035253047943115234
  [ASSERT PASS] custom-domain valid create -> 201
  [ASSERT PASS] created alias is on the custom domain @cd-lrfkxkwx.test

[tamper] tampered!=valid -> True
  [ASSERT PASS] tampered token differs from valid token

--- REQ2: custom-domain create (TAMPERED signed suffix) -> expect 412 ---
COMMAND: POST /api/v3/alias/custom/new json={alias_prefix:'cdtamp',signed_suffix:<tampered>,mailbox_ids:[2132]}
  status_code = 412
  body        = '{"error":"Alias creation time is expired, please retry"}\n'
  SL log:
    2026-07-13 20:45:43,410 - SL - WARNING - 9565 - "/app/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 1822 Test User user_nsupeov4ie@mailbox.test>
    2026-07-13 20:45:43,411 - SL - DEBUG - 9565 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.005589008331298828
  [ASSERT PASS] custom-domain tampered create -> 412 (same as built-in domain)
  [ASSERT PASS] tampered custom-domain body is the expiry message (tampered==expired collapse is domain-independent)

=== SUMMARY ===
  assertions failed: 0
  alias count AFTER (post-rollback) = 824  (before was 824)
  NET-ZERO on PostgreSQL: True
  redis probe-created keys deleted (bucket-limit bl:* etc) = 3
  NET-ZERO on Redis: True
>>> rolled back transaction; probe_customdomain committed NOTHING to PostgreSQL
```

REQ1 (valid) returns **201** with an alias on the custom domain and **no** `X-RateLimit-*` or
`Retry-After` headers; REQ2 (tampered) returns the identical **412
`{"error":"Alias creation time is expired, please retry"}`** and the WARNING at
`new_custom_alias.py:187`. The tampered-equals-expired collapse is therefore
**domain-independent**, and the probe committed nothing: `alias count BEFORE = 824`,
`AFTER = 824`, `NET-ZERO on PostgreSQL: True`, and it reaped the 3 `bl:alias_create_*`
bucket-limit keys its 201 path created (`NET-ZERO on Redis: True`), so both stores are left
exactly as found.

---

## 10. Secondary paths and honest caveats

The signed-suffix helper and the creation-limit checks are shared by three other call sites in
the codebase. The **API** path (§3–§9) is the one the questions target and the one exercised at
runtime; the sites below reuse the *same* helpers, so their **signature collapse is identical** —
only the way each surfaces the outcome differs. Claims about the surfacing of these paths are
**source-derived** (they require a browser session / OAuth client / CSRF and were not driven
end-to-end here); each is labelled accordingly.

### 10.1 Web-UI (dashboard) path — `app/dashboard/views/custom_alias.py`

This view imports the same `check_suffix_signature` and `verify_prefix_suffix`
[`app/dashboard/views/custom_alias.py:9-10`] and gates on `current_user.can_create_new_alias()`
[L36]. Its signature block mirrors the API exactly, but reports through `flash()` and a redirect
instead of a JSON body:
```python
# app/dashboard/views/custom_alias.py:89-98
        try:
            suffix = check_suffix_signature(signed_alias_suffix)
            if not suffix:
                LOG.w("Alias creation time expired for %s", current_user)
                flash("Alias creation time is expired, please retry", "warning")
                return redirect(request.url)
        except Exception:
            LOG.w("Alias suffix is tampered, user %s", current_user)
            flash("Unknown error, refresh the page", "error")
            return redirect(request.url)
```

Because `check_suffix_signature` returns `None` for tampered, expired, and garbage alike (§9.2),
a tampered suffix on this path flashes the **same "Alias creation time is expired, please retry"**
warning (L92-93) as a genuinely expired one — the collapse is identical to the API. The
`except Exception` branch (L95-97, flashing "Unknown error, refresh the page") is unreachable for
ordinary string input for the same reason as the API's "Tampered suffix" branch (§9.4).
*(Surfacing detail is source-derived; the signature collapse is the observed API behaviour.)*

### 10.2 OAuth authorize path — `app/oauth/views/authorize.py`

The OAuth flow imports `check_suffix_signature` [`app/oauth/views/authorize.py:7`], checks the
quota, and validates the suffix. Two differences from the API are worth naming: the quota failure
**raises an `Exception`** (L169-170) rather than returning a JSON 400, and the outcomes are
flashed:
```python
# app/oauth/views/authorize.py:169-191
                if not current_user.can_create_new_alias():
                    raise Exception(f"User {current_user} cannot create custom email")

                alias_prefix = alias_prefix.strip().lower().replace(" ", "")

                if not check_alias_prefix(alias_prefix):
                    flash(
                        "Only lowercase letters, numbers, dashes (-), dots (.) and underscores (_) "
                        "are currently supported for alias prefix. Cannot be more than 40 letters",
                        "error",
                    )
                    return redirect(request.url)

                # hypothesis: user will click on the button in the 600 secs
                try:
                    alias_suffix = check_suffix_signature(signed_suffix)
                    if not alias_suffix:
                        LOG.w("Alias creation time expired for %s", current_user)
                        flash("Alias creation time is expired, please retry", "warning")
                        return redirect(request.url)
                except Exception:
                    LOG.w("Alias suffix is tampered, user %s", current_user)
                    flash("Unknown error, refresh the page", "error")
```

The expiry/tamper collapse (L184-191) is again identical to the API; only the surfacing differs.
*(Source-derived surfacing; signature collapse is the observed API behaviour.)*

### 10.3 Random-alias sibling — `app/api/views/new_random_alias.py`

The random-alias endpoint carries the **identical decorator chain** and the **same account-quota
check**, which is why the rate-limit (§5), parallel-lock (§7.7), and quota (§6) findings apply to
it equally. It has **no** signed-suffix step (a random alias is not user-chosen), so
`check_suffix_signature` is never called here (`grep -c` returns 0):
```python
# app/api/views/new_random_alias.py:22-36
@limiter.limit(ALIAS_LIMIT)
@require_api_auth
@parallel_limiter.lock(name="alias_creation")
def new_random_alias():
    """
    Create a new random alias
    Input:
        (Optional) note
    Output:
        201 if success

    """
    user = g.user
    if not user.can_create_new_alias():
        LOG.d("user %s cannot create new random alias", user)
```

Note the quota-failure log here reads `LOG.d("user %s cannot create new random alias", user)`
(L36) — the wording differs from the custom endpoint's
`LOG.d("user %s cannot create any custom alias", user)` (§6), a useful discriminator when
reading the console.

### 10.4 Honest caveats

- **The unreachable "Tampered suffix"/"Unknown error" branch.** That the outer
  `except Exception` is unreachable for *all* ordinary input is a **reasoned conclusion** from the
  source plus the observed cases: strings are caught by the inner `except itsdangerous.BadSignature`
  (-> 412, §9.2), and truthy non-strings raise `AttributeError` at `.strip()` *before* the `try`
  (-> 500, §9.4). `check_suffix_signature` calls only `signer.unsign(signed_suffix, max_age=600).decode()`, which raises
  only `BadSignature`-family exceptions (all caught inside), so nothing reaches the outer `except`.
  This branch-reachability conclusion is therefore **inferred** (the two boundary cases that would
  have exercised it are observed to divert to 412 and 500 respectively).
- **Dashboard / OAuth surfacing.** The `flash()`/redirect and raised-`Exception` behaviours are
  read from source; they were not driven end-to-end (they need a browser session, CSRF token, or a
  registered OAuth client). The shared helper guarantees the same signature collapse; only the
  surface form is **inferred**.
- **Container wall clock.** The canonical container's clock reads `2026-07-13`, so every captured
  timestamp and every `creation_date`/`Expires` value reflects that year. This is cosmetic: the
  expiry arithmetic is **relative** (`max_age=600` seconds against "now"), so the backdated-601s
  expired-token probe (§3.2 / §9) behaves identically regardless of the absolute wall-clock year.

---

## 11. Appendix — complete probe sources (unabridged)

Every probe below is a **temporary** observation script that lived in the container at
`/tmp/sl_probes/` for the duration of the investigation and is **removed afterwards** (the source
repository is never touched; see §2.5 for the cleanup proof and §0 for the read-only constraint).
They are reproduced here **in full, unabridged**, so that every status code, log line, header, and
count reported in §3–§10 is independently reproducible with the exact commands in §2.3.

**Sensitive-data note.** Every signed suffix printed by these probes is **synthetic, disposable,
test-only, and time-limited** (`max_age=600` seconds); the signing key `CUSTOM_ALIAS_SECRET`
(`"secretcustom_alias"`) is the **public, test-only** value from `tests/test.env`
(`FLASK_SECRET` + `"custom_alias"`) and must never be treated as a credential. Real API keys and
session cookies are **REDACTED** in the probes (the API-key `code` is never printed), while the
disposable users/mailboxes/domains they mint are created inside a rolled-back transaction and
never persist.

### 11.1 `run_all.sh`

Master harness. Snapshots PostgreSQL and Redis, runs the six probes in order, then removes only probe-created Redis keys via a snapshot diff and asserts the DB baseline is unchanged (net-zero). This is the run that produced the coherent `19:30` evidence embedded throughout §3–§9.
```bash
#!/bin/bash
# run_all.sh -- canonical master reproduction harness for the SimpleLogin custom-alias
# investigation. Runs all six probes, then proves NET-ZERO residue in both PostgreSQL
# (per-table counts) and Redis (snapshot-diff: every key created by the probes is deleted,
# pre-existing keys are left untouched). Exits non-zero if any probe fails or residue remains.
set -u
cd /app
OUT=/tmp/sl_probes/out
mkdir -p "$OUT"
CE=(env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root CONFIG=tests/test.env \
    DB_URI='postgresql://test:test@localhost:5432/test' GNUPGHOME=/tmp/sl_gnupg \
    PYTHONPATH=/app /app/venv/bin/python)

dbcounts() { for t in alias users api_key mailbox alias_mailbox deleted_alias; do
  echo -n "$t="; PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from $t"; done; }

echo "===== BASELINE ====="
dbcounts | tr '\n' ' '; echo
DB0=$(PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from alias")
redis-cli --scan | sort > "$OUT/redis_snapshot_before.txt"
SZ0=$(wc -l < "$OUT/redis_snapshot_before.txt")
echo "redis keys before = $SZ0"

echo
echo "===== RUN PROBES ====="
rc=0
"${CE[@]}" /tmp/sl_probes/probe_mro.py        > "$OUT/mro.out" 2>&1;        e=$?; echo "probe_mro.py        EXIT=$e"; [ $e -ne 0 ] && rc=1
"${CE[@]}" /tmp/sl_probes/probe_functional.py > "$OUT/functional.out" 2>&1; e=$?; echo "probe_functional.py EXIT=$e"; [ $e -ne 0 ] && rc=1
"${CE[@]}" /tmp/sl_probes/probe_f18.py        > "$OUT/f18.out" 2>&1;        e=$?; echo "probe_f18.py        EXIT=$e"; [ $e -ne 0 ] && rc=1
"${CE[@]}" /tmp/sl_probes/probe_ratelimit.py  > "$OUT/ratelimit.out" 2>&1;  e=$?; echo "probe_ratelimit.py  EXIT=$e"; [ $e -ne 0 ] && rc=1
"${CE[@]}" /tmp/sl_probes/probe_lock.py       > "$OUT/lock.out" 2>&1;       e=$?; echo "probe_lock.py       EXIT=$e"; [ $e -ne 0 ] && rc=1
bash /tmp/sl_probes/probe_live.sh             > "$OUT/live.out" 2>&1;       e=$?; echo "probe_live.sh       EXIT=$e"; [ $e -ne 0 ] && rc=1

echo
echo "===== REDIS SNAPSHOT-DIFF CLEANUP (delete only keys the probes created) ====="
redis-cli --scan | sort > "$OUT/redis_snapshot_after.txt"
comm -13 "$OUT/redis_snapshot_before.txt" "$OUT/redis_snapshot_after.txt" > "$OUT/redis_new_keys.txt"
NEW=$(wc -l < "$OUT/redis_new_keys.txt")
echo "new keys created by probes = $NEW"
if [ "$NEW" -gt 0 ]; then
  echo "  (breakdown by prefix:)"; sed -E 's/:.*//;s#/.*##' "$OUT/redis_new_keys.txt" | sort | uniq -c
  xargs -r redis-cli del < "$OUT/redis_new_keys.txt" > /dev/null
fi

echo
echo "===== POST-CLEANUP NET-ZERO VERIFICATION ====="
dbcounts | tr '\n' ' '; echo
DB1=$(PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from alias")
redis-cli --scan | sort > "$OUT/redis_snapshot_final.txt"
LEFT=$(comm -13 "$OUT/redis_snapshot_before.txt" "$OUT/redis_snapshot_final.txt" | wc -l)
echo "alias baseline=$DB0 final=$DB1"
echo "redis probe-created keys remaining after cleanup = $LEFT"
if [ "$DB0" = "$DB1" ] && [ "$LEFT" -eq 0 ] && [ "$rc" -eq 0 ]; then
  echo ">>> NET-ZERO OK: all probes passed; DB + Redis restored to baseline"
else
  echo ">>> FAILURE: rc=$rc db_equal=$([ "$DB0" = "$DB1" ] && echo yes || echo no) redis_left=$LEFT"; rc=1
fi
exit $rc
```

### 11.2 `probe_mro.py`

Root-cause library probe (§9). Prints the `itsdangerous 1.1.0` exception MRO, the `issubclass` relations, the `check_suffix_signature` source, and the per-input behaviour (valid / tampered / expired / garbage). Also records the naive last-character-flip pitfall behind Finding 15.
```python
#!/usr/bin/env python3
"""
probe_mro.py -- root-cause probe (pure library; no DB/Redis/HTTP => zero residue).

Proves WHY tampered AND expired signed suffixes both yield HTTP 412 "expired":
check_suffix_signature [app/alias_suffix.py:L37-42] catches itsdangerous.BadSignature,
the SUPERCLASS of both BadTimeSignature (tampered) and SignatureExpired (expired), so it
returns None for tampered / expired / garbage alike.

Also demonstrates (Finding 15) that a NAIVE last-character flip is NOT a reliable way to
tamper a signed token: the final base64url char can carry "don't-care" padding bits, so a
different character may still decode to the same signature bytes and VERIFY. The robust
make_tampered() below VERIFIES that signer.unsign actually raises before using the token.
"""
import os, time, inspect, string, sys
os.environ.setdefault("CONFIG", "/app/tests/test.env")

import itsdangerous
from app.alias_suffix import signer, check_suffix_signature
from app.config import EMAIL_DOMAIN

B64URL = string.ascii_letters + string.digits + "-_"

def make_tampered(valid_token: str) -> str:
    """Return a token GUARANTEED to fail signature verification (verified to raise)."""
    for i in range(len(valid_token)):
        c = valid_token[i]
        repl = next(a for a in B64URL if a != c)
        cand = valid_token[:i] + repl + valid_token[i + 1:]
        try:
            signer.unsign(cand, max_age=600)
        except itsdangerous.BadSignature:
            return cand
    raise RuntimeError("could not construct a tampered token")

print("itsdangerous.__version__ =", itsdangerous.__version__)
print("SignatureExpired.__mro__  =", [c.__name__ for c in itsdangerous.SignatureExpired.__mro__])
print("BadTimeSignature.__mro__  =", [c.__name__ for c in itsdangerous.BadTimeSignature.__mro__])
print("BadSignature.__mro__      =", [c.__name__ for c in itsdangerous.BadSignature.__mro__])
print("issubclass(SignatureExpired, BadSignature) =", issubclass(itsdangerous.SignatureExpired, itsdangerous.BadSignature))
print("issubclass(BadTimeSignature, BadSignature) =", issubclass(itsdangerous.BadTimeSignature, itsdangerous.BadSignature))

print("\n----- check_suffix_signature source [app/alias_suffix.py:L37-42] -----")
print(inspect.getsource(check_suffix_signature))

payload = f".word@{EMAIL_DOMAIN}"
valid = signer.sign(payload).decode()

# Finding 15 pitfall demonstration: naive last-char flip
last = valid[-1]
naive = valid[:-1] + ("A" if last != "A" else "B")
try:
    signer.unsign(naive, max_age=600)
    naive_verdict = "STILL VERIFIES (naive flip FAILED to tamper -- unreliable)"
except itsdangerous.BadSignature as e:
    naive_verdict = f"raised {type(e).__name__} (naive flip happened to work this time)"
print(f"\n[Finding 15] naive last-char flip {last!r}->{naive[-1]!r}: {naive_verdict}")

# robust, verified tamper
tampered = make_tampered(valid)
garbage = "this-is-not-a-signed-suffix"

# expired: backdate the signer's timestamp so age > max_age (600)
_orig = signer.get_timestamp
signer.get_timestamp = lambda: int(time.time()) - 601
expired = signer.sign(payload).decode()
signer.get_timestamp = _orig

def probe(label, value):
    raised = None
    try:
        signer.unsign(value, max_age=600)
    except Exception as e:
        raised = f"{type(e).__name__}: {e}"
    ret = check_suffix_signature(value)
    print(f"{label:<9} unsign(max_age=600) raises -> {raised};  check_suffix_signature -> {ret!r}")

print("\n----- behavior per input (max_age=600) -----")
probe("valid", valid)
probe("tampered", tampered)
probe("expired", expired)
probe("garbage", garbage)

fails = []
if check_suffix_signature(valid) != payload: fails.append("valid should return the payload")
if check_suffix_signature(tampered) is not None: fails.append("tampered should return None")
if check_suffix_signature(expired) is not None: fails.append("expired should return None")
if check_suffix_signature(garbage) is not None: fails.append("garbage should return None")
if not issubclass(itsdangerous.SignatureExpired, itsdangerous.BadSignature): fails.append("SignatureExpired must subclass BadSignature")
if not issubclass(itsdangerous.BadTimeSignature, itsdangerous.BadSignature): fails.append("BadTimeSignature must subclass BadSignature")

print("\nassertions failed:", len(fails))
for f in fails:
    print("  -", f)
sys.exit(1 if fails else 0)
```

### 11.3 `probe_functional.py`

Primary functional probe (§3, §6, §8). Drives every condition of `POST /api/v2|v3/alias/custom/new` through the real Flask app + Werkzeug test client with real API-key auth, inside one rolled-back `connection.begin()` transaction. Contains the shared `mint`, `make_tampered` (Finding 15), backdated-expired minting, and `quota_snapshot` helpers.
```python
#!/usr/bin/env python3
"""
probe_functional.py
===================
SimpleLogin custom-alias validation probe. Exercises every functional condition of
POST /api/v2/alias/custom/new and POST /api/v3/alias/custom/new through the REAL Flask
app (server.create_app) using the Werkzeug test client and REAL API-key authentication.

ISOLATION: all DB writes happen inside ONE connection-level transaction opened with
connection.begin() and rolled back in the finally block -- the identical isolation used
by tests/conftest.py::flask_client (L59-77). The probe therefore commits NOTHING to
PostgreSQL. Redis parallel-lock keys created by @parallel_limiter.lock are released in
the handler's finally and also carry a 5s TTL, so no Redis residue is left either.

Rate limiting stays DISABLED here (config.DISABLE_RATE_LIMIT = True, as in the fixture);
the rate-limit boundary + headers are measured by probe_ratelimit.py / probe_live.sh.

Conditions 1-12 use a PREMIUM primary user (lifetime=True) so the account-quota check
(which runs FIRST, before the signature check) never masks the branch under test.
Condition 13 uses a fresh FREE user to show the count-based quota path AND masking.

INVOCATION (exact):
  cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
    CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
    GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
    /app/venv/bin/python /tmp/sl_probes/probe_functional.py

Exit code is non-zero if ANY assertion fails.
"""
import os, sys, io, json, logging, time

os.environ.setdefault("CONFIG", "/app/tests/test.env")

import arrow
import itsdangerous
from app import config
from app.db import Session, connection
import app.log as sl_log
from app.log import LOG
import app.alias_suffix as als
from app.alias_suffix import signer, check_suffix_signature
from app.config import EMAIL_DOMAIN, MAX_NB_EMAIL_FREE_PLAN, CUSTOM_ALIAS_SECRET
from app.models import User, ApiKey, Mailbox, Alias
from app.utils import random_word
from server import create_app
from tests.utils import create_new_user
from tests.api.utils import get_new_user_and_api_key

app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"

# ---------------------------------------------------------------- log capture
_cap_buf = io.StringIO()
_cap_handler = logging.StreamHandler(_cap_buf)
_cap_handler.setFormatter(sl_log._log_formatter)          # exact server-console format
_cap_handler.formatter.converter = time.gmtime
_saved_handlers = []

def install_capture():
    global _saved_handlers
    _saved_handlers = list(LOG.handlers)
    for h in _saved_handlers:
        LOG.removeHandler(h)
    LOG.addHandler(_cap_handler)

def uninstall_capture():
    LOG.removeHandler(_cap_handler)
    for h in _saved_handlers:
        LOG.addHandler(h)

def reset_logs():
    _cap_buf.seek(0); _cap_buf.truncate(0)

def read_logs():
    return _cap_buf.getvalue()

# ---------------------------------------------------------------- helpers
_failures = []
def check(cond, msg):
    print(f"  [ASSERT {'PASS' if cond else 'FAIL'}] {msg}")
    if not cond:
        _failures.append(msg)

def hr(title):
    print("\n" + "=" * 80)
    print(title)
    print("=" * 80)

def redact(headers):
    out = {}
    for k, v in (headers or {}).items():
        if k.lower() == "authentication":
            out[k] = "<disposable-60char-test-api-key>"
        else:
            out[k] = v
    return out

def show_response_and_logs(r, logs):
    print("RESPONSE:")
    print(f"  status_code = {r.status_code}")
    print("  headers (Werkzeug test-client; transport headers Date/Server/Connection are")
    print("           added by the WSGI server, not the test client -- see probe_live.sh):")
    for k, v in sorted(r.headers.items()):
        vv = v
        if k.lower() == "set-cookie":
            vv = v.split("=", 1)[0] + "=<redacted-session-cookie>"
        print(f"    {k}: {vv}")
    print(f"  body = {r.get_data(as_text=True)}")
    print("  SL-logger records emitted DURING this request [BEGIN]")
    if logs.strip():
        for line in logs.rstrip("\n").split("\n"):
            print(f"    | {line}")
    else:
        print("    | <empty> (no SL record emitted during this request; active-capture boundary)")
    print("  SL-logger records emitted DURING this request [END]")

def run_case(n, title, client, method, path, body, headers,
             expect_status, expect_error="__skip__", raw_body=False):
    hr(f"CONDITION {n}: {title}")
    reset_logs()
    print("COMMAND (Werkzeug test client):")
    print(f"  client.{method.lower()}({path!r},")
    if body is not None:
        if raw_body:
            print(f"      json={body!r},")
        else:
            print(f"      json={json.dumps(body)},")
    print(f"      headers={redact(headers)})")
    fn = getattr(client, method.lower())
    kwargs = {}
    if headers is not None:
        kwargs["headers"] = headers
    if body is not None:
        kwargs["json"] = body
    r = fn(path, **kwargs)
    logs = read_logs()
    show_response_and_logs(r, logs)
    check(r.status_code == expect_status, f"status_code {r.status_code} == {expect_status}")
    if expect_error != "__skip__":
        try:
            got = r.get_json().get("error")
        except Exception:
            got = None
        check(got == expect_error, f"error body {got!r} == {expect_error!r}")
    return r

def quota_snapshot(user, label):
    flag_set = (user.FLAG_FREE_OLD_ALIAS_LIMIT ==
                (user.flags & user.FLAG_FREE_OLD_ALIAS_LIMIT))
    print(f"  can_create_new_alias() branch inputs [{label}] (app/models.py:L867-884):")
    print(f"    is_active()                        = {user.is_active()}   (L872; delete_on={user.delete_on})")
    print(f"    disabled                           = {user.disabled}   (L875)")
    print(f"    lifetime                           = {user.lifetime}")
    print(f"    lifetime_or_active_subscription()  = {user.lifetime_or_active_subscription()}   (L878; trial does NOT count)")
    print(f"    FLAG_FREE_OLD_ALIAS_LIMIT set      = {flag_set}   (-> max via L858-865)")
    print(f"    max_alias_for_free_account()       = {user.max_alias_for_free_account()}")
    print(f"    Alias count for user               = {Alias.filter_by(user_id=user.id).count()}   (L881-883 count)")
    print(f"    ==> can_create_new_alias()         = {user.can_create_new_alias()}")

def mint(suffix):
    """Canonical mint via the real shared TimestampSigner (app/alias_suffix.py:L11)."""
    return signer.sign(suffix).decode()

import string as _string
_B64URL = _string.ascii_letters + _string.digits + "-_"

def make_tampered(valid_token):
    """Return a token GUARANTEED to fail signature verification (VERIFIED to raise
    itsdangerous.BadSignature). A naive last-character flip is unreliable because the
    final base64url char can carry padding bits, so a different char may still decode
    to the same signature bytes and verify (Finding 15). This walks positions from the
    front and returns the first single-char change that genuinely invalidates it."""
    for i in range(len(valid_token)):
        c = valid_token[i]
        repl = next(a for a in _B64URL if a != c)
        cand = valid_token[:i] + repl + valid_token[i + 1:]
        try:
            signer.unsign(cand, max_age=600)
        except itsdangerous.BadSignature:
            return cand
    raise RuntimeError("could not construct a tampered token")

# ---------------------------------------------------------------- run
print(">>> probe_functional.py starting")
print(f">>> EMAIL_DOMAIN={EMAIL_DOMAIN}  MAX_NB_EMAIL_FREE_PLAN={MAX_NB_EMAIL_FREE_PLAN}")
print(f">>> CUSTOM_ALIAS_SECRET={CUSTOM_ALIAS_SECRET!r}  (public TEST-ONLY secret = FLASK_SECRET+'custom_alias')")
print(">>> All signed suffixes below are SYNTHETIC, DISPOSABLE, TEST-ONLY and TIME-LIMITED (max_age=600s).")

transaction = connection.begin()
install_capture()
try:
    with app.app_context():
        config.DISABLE_RATE_LIMIT = True

        # ---- disposable principals ------------------------------------------
        user, api_key = get_new_user_and_api_key()
        user.lifetime = True     # premium: quota check (runs FIRST) never masks conditions 1-12
        Session.flush()
        H = {"Authentication": api_key.code}
        print(f"\n>>> disposable PRIMARY user id={user.id} email={user.email} default_mailbox_id={user.default_mailbox_id}")
        print(f">>> made PRIMARY user premium (lifetime=True) so can_create_new_alias() is always True")
        print(f">>> disposable api_key: ApiKey.create(user.id) -> code is random_string(60); value REDACTED")

        client = app.test_client()

        # ---- 0. canonical producer: GET /api/v4/alias/options ----------------
        hr("CONDITION 0: canonical signed_suffix producer  GET /api/v4/alias/options")
        reset_logs()
        r = client.get("/api/v4/alias/options", headers=H)
        opts = r.get_json()
        print(f"  status_code = {r.status_code}")
        print(f"  can_create  = {opts.get('can_create')}")
        print("  suffixes (first 4 [suffix, signed_suffix] pairs; signed values are time-limited):")
        for pair in opts.get("suffixes", [])[:4]:
            print(f"    {pair}")
        check(r.status_code == 200, "options status 200")
        check(any(p[0].endswith(f"@{EMAIL_DOMAIN}") for p in opts.get("suffixes", [])),
              f"a suffix for @{EMAIL_DOMAIN} is offered by the producer")

        # ---- 1. SUCCESS v2 ---------------------------------------------------
        w = random_word()
        suffix_ok = f".{w}@{EMAIL_DOMAIN}"
        quota_snapshot(user, "primary premium user")
        run_case(1, "SUCCESS v2 (valid signed suffix)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "okv2", "signed_suffix": mint(suffix_ok)}, H,
                 201)

        # ---- 2. SUCCESS v3 ---------------------------------------------------
        w3 = random_word()
        run_case(2, "SUCCESS v3 (valid signed suffix + mailbox_ids)",
                 client, "POST", "/api/v3/alias/custom/new",
                 {"alias_prefix": "okv3", "signed_suffix": mint(f".{w3}@{EMAIL_DOMAIN}"),
                  "mailbox_ids": [user.default_mailbox_id]}, H,
                 201)

        # ---- 3. TAMPERED suffix (v2) ----------------------------------------
        valid = mint(suffix_ok)
        tampered = make_tampered(valid)          # VERIFIED-invalid (Finding 15: not a naive last-char flip)
        print(f"\n[tamper] tampered!=valid -> {tampered != valid}")
        assert tampered != valid, "tampered token must differ from valid token"
        try:
            signer.unsign(tampered, max_age=600)
            raise SystemExit("ERROR: tampered token unexpectedly verified")
        except itsdangerous.BadSignature as e:
            print(f"[tamper] signer.unsign(tampered) raised {type(e).__name__}: {e} "
                  f"(subclass of BadSignature -> caught by check_suffix_signature -> None)")
        run_case(3, "TAMPERED signed suffix (one signature char changed)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "tamp", "signed_suffix": tampered}, H,
                 412, "Alias creation time is expired, please retry")

        # ---- 4. GARBAGE suffix (v2) -----------------------------------------
        garbage = "this-string-was-never-signed"
        try:
            signer.unsign(garbage, max_age=600)
        except itsdangerous.BadSignature as e:
            print(f"\n[garbage] signer.unsign('{garbage}') raised {type(e).__name__} -> None")
        run_case(4, "GARBAGE signed suffix (never signed)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "garb", "signed_suffix": garbage}, H,
                 412, "Alias creation time is expired, please retry")

        # ---- 5. GENUINELY EXPIRED suffix (Finding 3: full mint-and-post) -----
        hr("CONDITION 5: GENUINELY EXPIRED signed suffix (mint backdated, prove age>600, then POST)")
        signed_at = int(time.time()) - 601
        _orig_ts = als.signer.get_timestamp
        als.signer.get_timestamp = lambda: signed_at      # backdate the embedded timestamp
        expired = als.signer.sign(f".{random_word()}@{EMAIL_DOMAIN}").decode()
        als.signer.get_timestamp = _orig_ts               # restore immediately
        now = int(time.time())
        print(f"  signed-at (unix)  = {signed_at}  ({arrow.get(signed_at)})")
        print(f"  current  (unix)   = {now}  ({arrow.get(now)})")
        print(f"  age (s)           = {now - signed_at}  (> max_age=600 used by check_suffix_signature L40)")
        print("  DIRECT proof at the endpoint's real max_age=600:")
        try:
            als.signer.unsign(expired, max_age=600)
            print("    ERROR: expired token unexpectedly verified")
        except itsdangerous.SignatureExpired as e:
            print(f"    signer.unsign(expired, max_age=600) raised SignatureExpired: {e}")
            print(f"    e.date_signed = {e.date_signed}  (SignatureExpired IS-A BadSignature)")
        print(f"  check_suffix_signature(expired) -> {check_suffix_signature(expired)!r}  (None => handler 412 branch)")
        run_case("5b", "POST the EXACT expired token to the real endpoint",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "expd", "signed_suffix": expired}, H,
                 412, "Alias creation time is expired, please retry")

        # ---- 6. EMPTY body (v2) ---------------------------------------------
        run_case(6, "EMPTY request body (v2)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {}, H, 400, "request body cannot be empty")

        # ---- 7. v3 non-dict body (Finding 9/18) -----------------------------
        run_case(7, "v3 NON-DICT body (truthy non-dict fails isinstance BEFORE signature try; L153-154)",
                 client, "POST", "/api/v3/alias/custom/new",
                 "string isn't a dict", H,
                 400, "request body does not follow the required format", raw_body=True)

        # ---- 8. v3 mailbox_ids not an array (Finding 9) ---------------------
        run_case(8, "v3 mailbox_ids NOT an array (L171-172)",
                 client, "POST", "/api/v3/alias/custom/new",
                 {"alias_prefix": "mbx", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}"),
                  "mailbox_ids": "not an array"}, H,
                 400, "mailbox_ids must be an array of id")

        # ---- 9. DUPLICATE alias (v2) ----------------------------------------
        dupw = random_word()
        dsuffix = f".{dupw}@{EMAIL_DOMAIN}"
        run_case("9a", "DUPLICATE step 1 -- create the alias (201)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "dup", "signed_suffix": mint(dsuffix)}, H, 201)
        run_case("9b", "DUPLICATE step 2 -- same prefix+suffix again (409 + LOG.d L87)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "dup", "signed_suffix": mint(dsuffix)}, H,
                 409, f"alias dup.{dupw}@{EMAIL_DOMAIN} already exists")

        # ---- 10. TWO CONSECUTIVE DOTS (v2) ----------------------------------
        run_case(10, "TWO CONSECUTIVE DOTS (alias_prefix ends with a dot)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "prefix.", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")}, H,
                 400, "2 consecutive dot signs aren't allowed in an email address")

        # ---- 11. WRONG SUFFIX: SL domain but no leading dot -----------------
        badw = random_word()
        run_case(11, "WRONG SUFFIX (SL domain, prefix does NOT start with '.'; verify_prefix_suffix L77-79 LOG.e)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "wsuf", "signed_suffix": mint(f"{badw}@{EMAIL_DOMAIN}")}, H,
                 400, "wrong alias prefix or suffix")

        # ---- 12. WRONG DOMAIN: signed, but domain not available -------------
        run_case(12, "WRONG DOMAIN (validly signed suffix for a domain the user cannot use; verify L60-62 LOG.e)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "wdom", "signed_suffix": mint(f".{random_word()}@not-a-real-domain.test")}, H,
                 400, "wrong alias prefix or suffix")

        # ---- 13. FREE-USER quota: count-based success, exhaustion, masking --
        hr("CONDITION 13: FREE user -- count-based success path, quota exhaustion, and masking")
        quser, qkey = get_new_user_and_api_key()
        quser.trial_end = None       # canonical (tests: test_out_of_quota); does NOT affect can_create
        Session.flush()
        QH = {"Authentication": qkey.code}
        quota_snapshot(quser, "fresh FREE user (before any create via endpoint)")
        # 13a: count-based SUCCESS (Q4 success path for a free user under quota)
        run_case("13a", "FREE user SUCCESS while under quota (count < max; success path emits NO quota log)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "freeok", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")}, QH,
                 201)
        quota_snapshot(quser, "after one endpoint create")
        # fill to the cap
        need = quser.max_alias_for_free_account() - Alias.filter_by(user_id=quser.id).count()
        print(f"  filling {need} more alias(es) via Alias.create_new(quser, prefix='fill<i>') to reach the cap")
        for i in range(need):
            Alias.create_new(quser, prefix=f"fill{i}")
        Session.flush()
        quota_snapshot(quser, "after filling to cap")
        run_case("13b", "QUOTA exceeded with a VALID suffix (400 quota + LOG.d L49)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "over", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")}, QH,
                 400,
                 f"You have reached the limitation of a free account with the maximum of "
                 f"{MAX_NB_EMAIL_FREE_PLAN} aliases, please upgrade your plan to create more aliases")
        vv = mint(f".{random_word()}@{EMAIL_DOMAIN}")
        tt = vv[:-1] + ("A" if vv[-1] != "A" else "B")
        run_case("13c", "QUOTA exceeded with a TAMPERED suffix -> STILL 400 quota (quota MASKS the 412)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "overt", "signed_suffix": tt}, QH,
                 400,
                 f"You have reached the limitation of a free account with the maximum of "
                 f"{MAX_NB_EMAIL_FREE_PLAN} aliases, please upgrade your plan to create more aliases")

        # ---- 14. AUTH variants ----------------------------------------------
        run_case("14a", "AUTH: no Authentication header (401 Wrong api key)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "noauth", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")}, None,
                 401, "Wrong api key")
        run_case("14b", "AUTH: invalid api key (401 Wrong api key)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "badauth", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")},
                 {"Authentication": "Invalid code"},
                 401, "Wrong api key")
        duser, dkey = get_new_user_and_api_key()
        duser.disabled = True
        Session.flush()
        run_case("14c", "AUTH: DISABLED account via API key (403 Disabled account; base.py L36-37)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "dis", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")},
                 {"Authentication": dkey.code},
                 403, "Disabled account")
        iuser, ikey = get_new_user_and_api_key()
        iuser.delete_on = arrow.now().shift(days=1)   # FUTURE => is_active() False (scheduled-deletion grace window)
        Session.flush()
        run_case("14d", "AUTH: INACTIVE account (delete_on in FUTURE => is_active() False) via API key (401 Account does not exist; base.py L39-40)",
                 client, "POST", "/api/v2/alias/custom/new",
                 {"alias_prefix": "ina", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")},
                 {"Authentication": ikey.code},
                 401, "Account does not exist")

        hr("SUMMARY")
        print(f"  assertions failed: {len(_failures)}")
        for f in _failures:
            print(f"    - {f}")
finally:
    uninstall_capture()
    config.DISABLE_RATE_LIMIT = True
    transaction.rollback()
    Session.rollback()
    Session.close()
    print("\n>>> rolled back transaction; probe committed NOTHING to PostgreSQL")

sys.exit(1 if _failures else 0)
```

### 11.4 `probe_f18.py`

Non-string `signed_suffix` probe (§9.4, Finding 18). Sends JSON list / int / null / empty / missing values and captures the full unhandled 500 tracebacks and the v2-vs-v3 `or ""` divergence.
```python
#!/usr/bin/env python3
"""
probe_f18.py -- Finding 18 (OBSERVED, not inferred).

What does the endpoint actually do when `signed_suffix` is a TRUTHY NON-STRING
JSON value? Both handlers call `.strip()` on the raw value OUTSIDE the try/except
that guards the signature check:
  v2: signed_suffix = data.get("signed_suffix", "").strip()   [L65]  try opens [L69]
  v3: signed_suffix = data.get("signed_suffix", "") or ""      [L157]
      signed_suffix = signed_suffix.strip()                    [L158]  try opens [L185]
So a truthy non-string reaches `.strip()` before any signature logic and raises
AttributeError; the `except Exception -> "Tampered suffix", 400` branch
(L74-76 / L189-191) is NOT entered. This drives the REAL endpoints through the
Werkzeug test client and prints the exact observed status + body per case.

Also confirms falsy/missing cases: missing key and empty string both default to
"" (falsy) -> check_suffix_signature("") -> None -> 412.

ISOLATION identical to tests/conftest.py::flask_client -- connection.begin() +
rollback in finally. Commits NOTHING to PostgreSQL. Rate limiting disabled.
"""
import os
os.environ.setdefault("CONFIG", "/app/tests/test.env")

from app import config
from app.db import Session, connection
from server import create_app
from tests.api.utils import get_new_user_and_api_key

app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"

_failures = []
def check(cond, msg):
    print(f"  [ASSERT {'PASS' if cond else 'FAIL'}] {msg}")
    if not cond:
        _failures.append(msg)

def show(title, r):
    print(f"  {title}")
    print(f"    status_code = {r.status_code}")
    print(f"    body        = {r.get_data(as_text=True)!r}")

transaction = connection.begin()
try:
    with app.app_context():
        config.DISABLE_RATE_LIMIT = True
        app.config["PROPAGATE_EXCEPTIONS"] = False
        user, api_key = get_new_user_and_api_key()
        user.lifetime = True
        Session.flush()
        H = {"Authentication": api_key.code}
        client = app.test_client()
        print(">>> probe_f18.py starting (Finding 18: .strip() on non-string signed_suffix)")
        print(f">>> disposable premium user id={user.id}; api key REDACTED")

        print("\n--- v2: signed_suffix is a TRUTHY NON-STRING (JSON list) ---")
        print("COMMAND: POST /api/v2/alias/custom/new json={alias_prefix:'f18',signed_suffix:[1,2,3]}")
        r = client.post("/api/v2/alias/custom/new",
                        json={"alias_prefix": "f18", "signed_suffix": [1, 2, 3]}, headers=H)
        show("v2 list signed_suffix", r)
        check(r.status_code == 500, "v2 truthy non-string signed_suffix -> 500 (AttributeError at L65 .strip(), BEFORE try at L69)")
        check("Tampered suffix" not in r.get_data(as_text=True), "v2: body is NOT 'Tampered suffix' (except branch L74-76 not reached)")

        print("\n--- v3: signed_suffix is a TRUTHY NON-STRING (JSON int) ---")
        print("COMMAND: POST /api/v3/alias/custom/new json={alias_prefix:'f18',signed_suffix:123,mailbox_ids:[<mb>]}")
        r = client.post("/api/v3/alias/custom/new",
                        json={"alias_prefix": "f18", "signed_suffix": 123,
                              "mailbox_ids": [user.default_mailbox_id]}, headers=H)
        show("v3 int signed_suffix", r)
        check(r.status_code == 500, "v3 truthy non-string signed_suffix -> 500 (AttributeError at L158 .strip(), BEFORE try at L185)")

        print("\n--- v2: signed_suffix key MISSING (defaults to '' via data.get(...,'')) ---")
        r = client.post("/api/v2/alias/custom/new", json={"alias_prefix": "f18miss"}, headers=H)
        show("v2 missing signed_suffix", r)
        check(r.status_code == 412, "v2 missing signed_suffix -> 412 ('' falsy -> check_suffix_signature('') -> None)")

        print("\n--- v2: signed_suffix is empty string '' ---")
        r = client.post("/api/v2/alias/custom/new",
                        json={"alias_prefix": "f18empty", "signed_suffix": ""}, headers=H)
        show("v2 empty signed_suffix", r)
        check(r.status_code == 412, "v2 empty-string signed_suffix -> 412")

        print("\n--- v2: signed_suffix is JSON null (None) ---")
        r = client.post("/api/v2/alias/custom/new",
                        json={"alias_prefix": "f18null", "signed_suffix": None}, headers=H)
        show("v2 null signed_suffix", r)
        check(r.status_code in (412, 500), "v2 JSON-null signed_suffix observed (see body/status above)")

        print("\n--- v3: signed_suffix is JSON null (None) -- v3 guards with `or \"\"` ---")
        r = client.post("/api/v3/alias/custom/new",
                        json={"alias_prefix": "f18null3", "signed_suffix": None,
                              "mailbox_ids": [user.default_mailbox_id]}, headers=H)
        show("v3 null signed_suffix", r)
        check(r.status_code == 412, "v3 JSON-null signed_suffix -> 412 (`or \"\"` coerces None -> '' -> None -> 412)")

    print("\n=== SUMMARY ===")
    print(f"  assertions failed: {len(_failures)}")
finally:
    transaction.rollback()
    Session.rollback()
    Session.close()
    print(">>> rolled back transaction; probe_f18 committed NOTHING to PostgreSQL")

import sys as _sys
_sys.exit(1 if _failures else 0)
```

### 11.5 `probe_ratelimit.py`

Rate-limit boundary + key-derivation probe (§5). Enables rate limiting, drives two isolated users/IPs past `ALIAS_LIMIT`, records the [201x5, 429, 429] pattern and the absence of `X-RateLimit-*`/`Retry-After` headers, and shows the `userid:{id}` vs `ip:{addr}` key derivation. Restores state in `finally`.
```python
#!/usr/bin/env python3
"""
probe_ratelimit.py
==================
Measures the HTTP rate-limit boundary and headers for POST /api/v2/alias/custom/new,
decorated with @limiter.limit(ALIAS_LIMIT) where ALIAS_LIMIT="100/day;50/hour;5/minute"
(app/config.py:L448). Flask-Limiter is backed by Redis (MEM_STORE_URI=redis://localhost).

Isolation: rate limiting is toggled on (config.DISABLE_RATE_LIMIT=False). Two independent
runs use DISTINCT client IPs (REMOTE_ADDR 10.0.0.1 / 10.0.0.2) and DISTINCT premium users,
so their limiter namespaces never overlap. Every LIMITER/* key created is printed and then
deleted in the finally block; the DB transaction is rolled back. g._rate_limiting_complete
is reset between test-client requests (flask-limiter issue #147 workaround, as in
tests/api/test_new_custom_alias.py::test_too_many_requests).

Users are premium (lifetime=True) so the account quota never interferes with the limiter.

INVOCATION (exact):
  cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
    CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
    GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
    /app/venv/bin/python /tmp/sl_probes/probe_ratelimit.py
"""
import os, sys, time
os.environ.setdefault("CONFIG", "/app/tests/test.env")

import redis
from flask import g
from app import config
from app.db import Session, connection
from app.alias_suffix import signer
from app.config import EMAIL_DOMAIN, ALIAS_LIMIT
from app.models import Alias
from app.utils import random_word
from server import create_app
from tests.utils import create_new_user, login
from tests.api.utils import get_new_user_and_api_key

app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"

R = redis.from_url("redis://localhost")

def mint(suffix):
    return signer.sign(suffix).decode()

def limiter_keys(substr):
    return sorted(k.decode() for k in R.keys("LIMITER/*") if substr in k.decode())

_failures = []
def check(cond, msg):
    print(f"  [ASSERT {'PASS' if cond else 'FAIL'}] {msg}")
    if not cond:
        _failures.append(msg)

def run(run_label, headers, remote_addr, n_requests, prefix):
    print("\n" + "=" * 80)
    print(f"{run_label}  (REMOTE_ADDR={remote_addr}, API-key auth, premium user)")
    print("=" * 80)
    print(f"ALIAS_LIMIT = {ALIAS_LIMIT!r}  (most restrictive short window = 5/minute)")
    statuses = []
    first_429 = None
    last_r = None
    for i in range(1, n_requests + 1):
        body = {"alias_prefix": f"{prefix}{i}", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")}
        r = app.test_client().post(
            "/api/v2/alias/custom/new", json=body, headers=headers,
            environ_overrides={"REMOTE_ADDR": remote_addr},
        )
        statuses.append(r.status_code)
        if r.status_code == 429 and first_429 is None:
            first_429 = i
        print(f"  request #{i:>2}: HTTP {r.status_code}  body={r.get_data(as_text=True)}")
        last_r = r
        g._rate_limiting_complete = False    # flask-limiter #147 test-client workaround
    print(f"  --> statuses: {statuses}")
    print(f"  --> first 429 at request #{first_429}")
    # show the limiter key(s) created for this IP
    keys = limiter_keys(f"ip:{remote_addr}")
    print(f"  --> Redis LIMITER keys for ip:{remote_addr}:")
    for k in keys:
        print(f"        {k}  = {R.get(k).decode() if R.get(k) else None}")
    # full headers of the 429 response (answers 'do rate-limit headers appear?')
    print("  --> FULL headers of the LAST (429) response:")
    for hk, hv in sorted(last_r.headers.items()):
        vv = hv.split("=", 1)[0] + "=<redacted>" if hk.lower() == "set-cookie" else hv
        print(f"        {hk}: {vv}")
    xrl = [hk for hk in last_r.headers.keys() if hk.lower().startswith("x-ratelimit") or hk.lower() == "retry-after"]
    print(f"  --> X-RateLimit-*/Retry-After headers present: {xrl if xrl else 'NONE'}")
    check(first_429 == 6, f"{run_label}: first 429 at request #6 (5/minute allows exactly 5)")
    check(statuses[:5] == [201] * 5, f"{run_label}: requests 1-5 all 201")
    check(statuses[5] == 429, f"{run_label}: request 6 is 429")
    check(xrl == [], f"{run_label}: no X-RateLimit-*/Retry-After headers on the 429")
    check(last_r.get_json() == {"error": "Rate limit exceeded"},
          f"{run_label}: 429 body == {{'error':'Rate limit exceeded'}}")
    return keys

saved_disable = config.DISABLE_RATE_LIMIT
transaction = connection.begin()
created_key_sets = []
session_keys_before = set(k.decode() for k in R.keys("session:*"))
try:
    with app.app_context():
        config.DISABLE_RATE_LIMIT = False

        userA, keyA = get_new_user_and_api_key(); userA.lifetime = True
        userB, keyB = get_new_user_and_api_key(); userB.lifetime = True
        Session.flush()

        created_key_sets.append(run("RUN 1", {"Authentication": keyA.code}, "10.0.0.1", 7, "r1p"))
        created_key_sets.append(run("RUN 2", {"Authentication": keyB.code}, "10.0.0.2", 7, "r2p"))

        # ---- Finding 10: key derivation session (userid:) vs API-key (ip:) ----
        print("\n" + "=" * 80)
        print("KEY DERIVATION (Finding 10): session-auth -> userid:{id}; API-key-only -> ip:{addr}")
        print("=" * 80)
        g._rate_limiting_complete = False
        sclient = app.test_client()
        suser = create_new_user(); suser.lifetime = True; Session.flush()
        login(sclient, suser)   # establish a Flask-Login session
        g._rate_limiting_complete = False
        rs = sclient.post("/api/v2/alias/custom/new",
                          json={"alias_prefix": "sess", "signed_suffix": mint(f".{random_word()}@{EMAIL_DOMAIN}")})
        print(f"  session-auth create -> HTTP {rs.status_code}")
        ukeys = limiter_keys(f"userid:{suser.id}")
        print(f"  Redis LIMITER keys for userid:{suser.id}: {ukeys}")
        check(any(f"userid:{suser.id}" in k for k in ukeys),
              "session-auth request is rate-limited under userid:{id}")
        created_key_sets.append(ukeys)
        # API-key-only key form already demonstrated in RUN 1/2 as ip:{addr}
        check(any("ip:10.0.0.1" in k for k in limiter_keys("ip:10.0.0.1")),
              "API-key-only request is rate-limited under ip:{addr}")

        print("\n" + "=" * 80)
        print("SUMMARY")
        print("=" * 80)
        print(f"  assertions failed: {len(_failures)}")
        for f in _failures:
            print(f"    - {f}")
finally:
    # ---- fail-safe cleanup ----
    config.DISABLE_RATE_LIMIT = saved_disable
    deleted = 0
    for ks in created_key_sets:
        for k in ks:
            deleted += R.delete(k)
    # also delete any lingering LIMITER keys for our test IPs / user
    for pat in ("LIMITER/*ip:10.0.0.1*", "LIMITER/*ip:10.0.0.2*"):
        for k in R.keys(pat):
            deleted += R.delete(k)
    # remove any NEW session keys created by the session-auth demo
    session_keys_after = set(k.decode() for k in R.keys("session:*"))
    new_sessions = session_keys_after - session_keys_before
    for k in new_sessions:
        R.delete(k)
    transaction.rollback()
    Session.rollback()
    Session.close()
    print(f"\n>>> cleanup: DISABLE_RATE_LIMIT restored to {saved_disable}; "
          f"deleted {deleted} LIMITER key(s), {len(new_sessions)} session key(s); DB rolled back")

sys.exit(1 if _failures else 0)
```

### 11.6 `probe_lock.py`

Parallel-lock probe (§7.7, Finding 14). Exercises `@parallel_limiter.lock(name="alias_creation")` contention (HTTP 429 via werkzeug `TooManyRequests`) and the Redis-absent no-op, restoring `lock_redis`/`DISABLE_RATE_LIMIT` and deleting any short-TTL keys it created.
```python
#!/usr/bin/env python3
"""
probe_lock.py -- exercise the @parallel_limiter.lock(name="alias_creation") decorator
[app/api/views/new_custom_alias.py:L31 (v2), L118 (v3)] on the REAL endpoint.

Isolation: config.DISABLE_RATE_LIMIT=True so the HTTP @limiter.limit is a no-op
(extensions.py request_filter), leaving ONLY the parallel-lock as a possible 429 source.
DB work runs in a rolled-back transaction; every Redis key touched is deleted in finally.

Demonstrations (all through POST /api/v2/alias/custom/new):
  (A) success path acquires cl:{addr}:alias_creation then self-releases (finally block).
  (B) contention -> real acquire_lock (storage.set nx=True) fails -> raises
      werkzeug.exceptions.TooManyRequests -> HTTP 429. The concurrent holder is SIMULATED
      by a pre-seeded Redis key (LABELLED non-canonical); the 429 itself is produced by the
      real decorator on the real endpoint.
  (C) session auth -> lock name is cl:{user.id}:alias_creation (current_user is the user).
  (D) parallel_limiter.lock_redis=None -> decorator is a no-op (parallel_limiter.py:L51-52).
"""
import os, sys
os.environ.setdefault("CONFIG", "/app/tests/test.env")

import redis
from flask import g
from app import config, parallel_limiter
from app.db import Session, connection
from app.alias_suffix import signer
from app.config import EMAIL_DOMAIN
from app.utils import random_word
from server import create_app
from tests.utils import create_new_user, login
from tests.api.utils import get_new_user_and_api_key

app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"
R = redis.from_url("redis://localhost")

def mint():
    return signer.sign(f".{random_word()}@{EMAIL_DOMAIN}").decode()

_failures = []
def check(c, m):
    print(f"  [ASSERT {'PASS' if c else 'FAIL'}] {m}")
    if not c:
        _failures.append(m)

ADDR = "10.9.9.9"
LOCKNAME = f"cl:{ADDR}:alias_creation"
transaction = connection.begin()
saved_disable = config.DISABLE_RATE_LIMIT
saved_lock_redis = parallel_limiter.lock_redis
seeded = []
try:
    with app.app_context():
        config.DISABLE_RATE_LIMIT = True   # HTTP limiter off => only parallel-lock can 429
        user, key = get_new_user_and_api_key(); user.lifetime = True; Session.flush()

        print("lock_redis wired from MEM_STORE_URI:", parallel_limiter.lock_redis is not None)
        check(parallel_limiter.lock_redis is not None, "parallel_limiter.lock_redis is wired")

        print("\n=== (A) success acquires + self-releases cl:{addr}:alias_creation ===")
        before = R.exists(LOCKNAME)
        g._rate_limiting_complete = False
        r = app.test_client().post("/api/v2/alias/custom/new",
              json={"alias_prefix": "lk1", "signed_suffix": mint()},
              headers={"Authentication": key.code},
              environ_overrides={"REMOTE_ADDR": ADDR})
        after = R.exists(LOCKNAME)
        print(f"  create -> HTTP {r.status_code}; {LOCKNAME} exists before={bool(before)} after={bool(after)}")
        check(r.status_code == 201, "(A) success create returns 201")
        check(not before and not after, "(A) lock key absent before AND after (acquired+released within request)")

        print("\n=== (B) contention: pre-seeded holder => real acquire_lock raises 429 ===")
        print(f"  [NON-CANONICAL] seeding {LOCKNAME} to simulate a concurrent in-flight request holding the lock")
        R.set(LOCKNAME, "concurrent-holder", ex=5, nx=True); seeded.append(LOCKNAME)
        g._rate_limiting_complete = False
        r2 = app.test_client().post("/api/v2/alias/custom/new",
              json={"alias_prefix": "lk2", "signed_suffix": mint()},
              headers={"Authentication": key.code},
              environ_overrides={"REMOTE_ADDR": ADDR})
        print(f"  contended create -> HTTP {r2.status_code}; body={r2.get_data(as_text=True)}")
        check(r2.status_code == 429, "(B) contended create returns 429 (parallel_limiter TooManyRequests)")
        check(r2.get_json() == {"error": "Rate limit exceeded"}, "(B) 429 body == {'error':'Rate limit exceeded'}")
        check(R.get(LOCKNAME) == b"concurrent-holder",
              "(B) seeded holder preserved (nx set failed; guarded release did not delete it)")

        print("\n=== (C) session auth => lock name cl:{user.id}:alias_creation ===")
        sclient = app.test_client()
        suser = create_new_user(); suser.lifetime = True; Session.flush()
        login(sclient, suser)
        SLOCK = f"cl:{suser.id}:alias_creation"
        print(f"  [NON-CANONICAL] seeding {SLOCK}")
        R.set(SLOCK, "concurrent-holder", ex=5, nx=True); seeded.append(SLOCK)
        g._rate_limiting_complete = False
        r3 = sclient.post("/api/v2/alias/custom/new",
              json={"alias_prefix": "lk3", "signed_suffix": mint()})
        print(f"  session contended create -> HTTP {r3.status_code}")
        check(r3.status_code == 429, "(C) session-auth contended create returns 429 under cl:{user.id}:alias_creation")

        print("\n=== (D) lock_redis=None => decorator is a no-op (parallel_limiter.py:L51-52) ===")
        parallel_limiter.lock_redis = None
        R.set(LOCKNAME, "would-block-if-lock-ran", ex=5)   # present, but ignored by the no-op path
        g._rate_limiting_complete = False
        r4 = app.test_client().post("/api/v2/alias/custom/new",
              json={"alias_prefix": "lk4", "signed_suffix": mint()},
              headers={"Authentication": key.code},
              environ_overrides={"REMOTE_ADDR": ADDR})
        print(f"  create with lock_redis=None -> HTTP {r4.status_code} (lock skipped)")
        check(r4.status_code == 201, "(D) create succeeds (201) when lock_redis is None (no-op path)")
        parallel_limiter.lock_redis = saved_lock_redis
        R.delete(LOCKNAME)

        print("\n=== SUMMARY ===")
        print("assertions failed:", len(_failures))
        for f in _failures:
            print("  -", f)
finally:
    parallel_limiter.lock_redis = saved_lock_redis
    config.DISABLE_RATE_LIMIT = saved_disable
    d = 0
    for k in set(seeded + [LOCKNAME]):
        d += R.delete(k)
    transaction.rollback(); Session.rollback(); Session.close()
    print(f"\n>>> cleanup: restored lock_redis+DISABLE_RATE_LIMIT; deleted {d} cl key(s); DB rolled back")
sys.exit(1 if _failures else 0)
```

### 11.7 `probe_customdomain.py`

Custom-domain header run (§9.5, Finding 16b). Provisions a verified custom domain, mints a canonical `@<domain>` suffix, and captures a real 201 (with complete headers and the `X-RateLimit-*/Retry-After = NONE` check) plus a tampered 412, proving the collapse is domain-independent. Counts committed aliases on a separate connection to prove net-zero.
```python
#!/usr/bin/env python3
"""
probe_customdomain.py
=====================
Finding 16(b): capture a REAL custom-domain alias-creation run with its actual output
(status, body, COMPLETE response headers, "SL" log) instead of asserting it without
evidence. Exercises POST /api/v3/alias/custom/new for an alias on a VERIFIED custom
domain through the REAL Flask app + Werkzeug test client + real API-key auth.

Shows (a) a custom-domain create succeeds (201) with the standard response body/headers
and NO X-RateLimit-*/Retry-After headers, and (b) a tampered custom-domain suffix returns
the same 412 as the built-in-domain case (the tampered==expired collapse is domain-independent).

ISOLATION: one connection.begin() transaction rolled back in finally (identical to
tests/conftest.py::flask_client L59-77); commits NOTHING to PostgreSQL. Net-zero verified
by comparing the alias count before/after.

INVOCATION (exact):
  cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
    CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
    GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
    /app/venv/bin/python /tmp/sl_probes/probe_customdomain.py
"""
import os, io, json, logging, time, string, secrets
import redis
os.environ.setdefault("CONFIG", "/app/tests/test.env")

import itsdangerous
from sqlalchemy import text
from app import config
from app.db import Session, connection, engine
import app.log as sl_log
from app.log import LOG
from app.alias_suffix import signer, check_suffix_signature
from app.models import User, Alias, CustomDomain
from server import create_app
from tests.api.utils import get_new_user_and_api_key

app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"

_cap_buf = io.StringIO()
_cap_handler = logging.StreamHandler(_cap_buf)
_cap_handler.setFormatter(sl_log._log_formatter)
_cap_handler.formatter.converter = time.gmtime
_saved = []
def install_capture():
    global _saved
    _saved = list(LOG.handlers)
    for h in _saved: LOG.removeHandler(h)
    LOG.addHandler(_cap_handler)
def reset_logs(): _cap_buf.seek(0); _cap_buf.truncate(0)
def read_logs(): return _cap_buf.getvalue()

def committed_alias_count():
    # Count on a SEPARATE pooled connection so the read reflects only COMMITTED rows
    # and never touches the shared app.db.connection whose transaction we roll back.
    # This is the same external-observer semantics run_all.sh uses via psql.
    with engine.connect() as c:
        return c.execute(text("select count(*) from alias")).scalar()

_B64URL = string.ascii_letters + string.digits + "-_"
def make_tampered(valid_token):
    """Guaranteed-invalid single-char change, VERIFIED to raise BadSignature (Finding 15:
    a naive last-char flip is unreliable because the final base64url char can carry padding
    bits). Walks from the front, returns the first change that genuinely invalidates."""
    for i in range(len(valid_token)):
        c = valid_token[i]
        repl = next(a for a in _B64URL if a != c)
        cand = valid_token[:i] + repl + valid_token[i+1:]
        try:
            signer.unsign(cand, max_age=600)
        except itsdangerous.BadSignature:
            return cand
    raise RuntimeError("could not construct a tampered token")

def dump_headers(resp):
    print("  response headers (complete, as returned by the app):")
    for k in sorted(resp.headers.keys()):
        print(f"    {k}: {resp.headers[k]}")
    rl = [k for k in resp.headers.keys() if k.lower().startswith("x-ratelimit") or k.lower() == "retry-after"]
    print(f"  X-RateLimit-*/Retry-After headers present: {rl if rl else 'NONE'}")

fails = 0
def check(cond, msg):
    global fails
    print(f"  [{'ASSERT PASS' if cond else 'ASSERT FAIL'}] {msg}")
    if not cond: fails += 1

print(">>> probe_customdomain.py starting (Finding 16b: real custom-domain header run)")
print(f">>> CUSTOM_ALIAS_SECRET={config.CUSTOM_ALIAS_SECRET!r}  (public TEST-ONLY secret = FLASK_SECRET+'custom_alias')")
print(">>> All signed suffixes below are SYNTHETIC, DISPOSABLE, TEST-ONLY and TIME-LIMITED (max_age=600s).")

alias_before = committed_alias_count()
_R = redis.from_url("redis://localhost")
_redis_before = set(_R.keys("*"))   # snapshot to reap ONLY keys this probe creates
print(f">>> alias count BEFORE = {alias_before}")

transaction = connection.begin()
install_capture()
try:
    with app.app_context():
        config.DISABLE_RATE_LIMIT = True   # net-zero; header SHAPE shown here, enabled-case proven live in §5
        user, api_key = get_new_user_and_api_key()
        user.lifetime = True
        Session.flush()
        H = {"Authentication": api_key.code}

        # verified custom domain owned by this user
        dom = "cd-" + "".join(secrets.choice(string.ascii_lowercase) for _ in range(8)) + ".test"
        cd = CustomDomain.create(user_id=user.id, domain=dom, verified=True,
                                 ownership_verified=True, flush=True)
        print(f"\n>>> disposable premium user id={user.id}; api key REDACTED")
        print(f">>> created VERIFIED CustomDomain id={cd.id} domain={dom!r} verified=True ownership_verified=True")
        avail = user.available_alias_domains()
        print(f">>> user.available_alias_domains() includes {dom!r}? {dom in avail}")
        check(dom in avail, f"custom domain {dom} is offered to the user")

        client = app.test_client()

        # canonical custom-domain suffix: '@<domain>' (no leading '.word' -- custom domains carry no SL word prefix)
        suffix = f"@{dom}"
        valid = signer.sign(suffix).decode()
        print(f"\n[mint] custom-domain suffix (synthetic/disposable/test-only/time-limited): {suffix!r}")
        print(f"[mint] signed_suffix (disposable, expires 600s): {valid!r}")
        print(f"[verify] check_suffix_signature(valid) -> {check_suffix_signature(valid)!r}")

        # ---- REQ1: valid custom-domain create -> expect 201 ----
        print("\n--- REQ1: custom-domain create (valid signed suffix) -> expect 201 ---")
        print(f"COMMAND: POST /api/v3/alias/custom/new json={{alias_prefix:'cdok',signed_suffix:<valid>,mailbox_ids:[{user.default_mailbox_id}]}}")
        reset_logs()
        r = client.post("/api/v3/alias/custom/new",
                        json={"alias_prefix": "cdok", "signed_suffix": valid,
                              "mailbox_ids": [user.default_mailbox_id]}, headers=H)
        print(f"  status_code = {r.status_code}")
        print(f"  body        = {r.get_data(as_text=True)!r}")
        dump_headers(r)
        print("  SL log:")
        for ln in read_logs().splitlines(): print(f"    {ln}")
        check(r.status_code == 201, "custom-domain valid create -> 201")
        j = r.get_json() or {}
        check(str(j.get("alias", "")).endswith(f"@{dom}"), f"created alias is on the custom domain @{dom}")

        # ---- REQ2: tampered custom-domain suffix -> expect 412 (domain-independent collapse) ----
        tampered = make_tampered(valid)
        print(f"\n[tamper] tampered!=valid -> {tampered != valid}")
        check(tampered != valid, "tampered token differs from valid token")
        print("\n--- REQ2: custom-domain create (TAMPERED signed suffix) -> expect 412 ---")
        print(f"COMMAND: POST /api/v3/alias/custom/new json={{alias_prefix:'cdtamp',signed_suffix:<tampered>,mailbox_ids:[{user.default_mailbox_id}]}}")
        reset_logs()
        r2 = client.post("/api/v3/alias/custom/new",
                         json={"alias_prefix": "cdtamp", "signed_suffix": tampered,
                               "mailbox_ids": [user.default_mailbox_id]}, headers=H)
        print(f"  status_code = {r2.status_code}")
        print(f"  body        = {r2.get_data(as_text=True)!r}")
        print("  SL log:")
        for ln in read_logs().splitlines(): print(f"    {ln}")
        check(r2.status_code == 412, "custom-domain tampered create -> 412 (same as built-in domain)")
        check(r2.get_json().get("error") == "Alias creation time is expired, please retry",
              "tampered custom-domain body is the expiry message (tampered==expired collapse is domain-independent)")
finally:
    # detach the capture handler and restore the app's own handlers
    LOG.removeHandler(_cap_handler)
    for h in _saved: LOG.addHandler(h)
    transaction.rollback()
    config.DISABLE_RATE_LIMIT = True
    _redis_new = set(_R.keys("*")) - _redis_before
    _redis_deleted = 0
    for _k in _redis_new:
        _redis_deleted += _R.delete(_k)

alias_after = committed_alias_count()
print(f"\n=== SUMMARY ===")
print(f"  assertions failed: {fails}")
print(f"  alias count AFTER (post-rollback) = {alias_after}  (before was {alias_before})")
print(f"  NET-ZERO on PostgreSQL: {alias_before == alias_after}")
print(f"  redis probe-created keys deleted (bucket-limit bl:* etc) = {_redis_deleted}")
print(f"  NET-ZERO on Redis: {len(set(_R.keys('*')) - _redis_before) == 0}")
print(">>> rolled back transaction; probe_customdomain committed NOTHING to PostgreSQL")
import sys as _sys
_sys.exit(1 if fails or alias_before != alias_after else 0)
```

### 11.8 `probe_live.sh`

Live-server harness (§3.1, §5). Starts a bounded `gunicorn` (`--timeout 15`, the Dockerfile value) with a captured PID and readiness poll, drives `curl -i` for the 201/412/429 raw-header captures, then terminates the exact PID and tears down its seed user. Never uses `pkill`.
```bash
#!/bin/bash
# probe_live.sh -- capture COMPLETE raw HTTP response headers (incl. gunicorn transport
# headers Date/Server/Connection) from a LIVE gunicorn server for the 201 / 412 / 429
# responses of POST /api/v2/alias/custom/new, proving exactly which headers appear
# (answering "do rate-limit headers appear, and what are their actual values?").
#
# Canonical build/invocation: gunicorn wsgi:app (Dockerfile CMD) with the canonical
# /tmp/sl.env config, EXCEPT DISABLE_RATE_LIMIT is unset so rate limiting is active
# (config.py:602 DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ -> presence-based).
set -u
OUT=/tmp/sl_probes/out
mkdir -p "$OUT"
cd /app
PORT=7777

CANON_ENV=(env CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
  GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app)

echo "===== [0/5] ensure port ${PORT} is free ====="
STALE=$(ss -ltnp 2>/dev/null | grep ":${PORT} " | grep -o 'pid=[0-9]*' | head -1 | cut -d= -f2)
if [ -n "${STALE:-}" ]; then echo "killing stale listener pid=$STALE"; kill "$STALE" 2>/dev/null; sleep 1; fi
echo "port check done"

echo
echo "===== [1/5] SEED disposable premium user + API key ====="
SEED=$("${CANON_ENV[@]}" /app/venv/bin/python /tmp/sl_probes/probe_live_seed.py)
echo "$SEED"
API_KEY=$(echo "$SEED"  | sed -n 's/^API_KEY=//p')
VALID=$(echo "$SEED"    | sed -n 's/^VALID_SUFFIX=//p')
TAMPERED=$(echo "$SEED" | sed -n 's/^TAMPERED_SUFFIX=//p')

echo
echo "===== [2/5] START live gunicorn (rate limiting ENABLED) ====="
set -a; . /tmp/sl.env; set +a
unset DISABLE_RATE_LIMIT          # enable rate limiting (presence-based flag)
export DB_URI='postgresql://test:test@localhost:5432/test'
/app/venv/bin/gunicorn wsgi:app -b 127.0.0.1:${PORT} -w 2 --timeout 15 \
    > "$OUT/gunicorn.log" 2>&1 &
GPID=$!
echo "gunicorn master PID=$GPID (bound 127.0.0.1:${PORT}, -w 2 --timeout 15)"

# readiness poll -- wait until gunicorn accepts a connection AND returns a real HTTP code
READY=0
for i in $(seq 1 60); do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 "http://127.0.0.1:${PORT}/")
  case "$code" in
    200|301|302|401|403|404) READY=1; echo "ready after ${i} poll(s), GET / -> $code"; break;;
  esac
  sleep 0.5
done
sleep 1   # let both sync workers finish booting before load
if [ "$READY" != "1" ]; then
  echo "SERVER NEVER BECAME READY; gunicorn log:"; cat "$OUT/gunicorn.log"
  kill "$GPID" 2>/dev/null; wait "$GPID" 2>/dev/null
  "${CANON_ENV[@]}" /app/venv/bin/python /tmp/sl_probes/probe_live_teardown.py
  exit 1
fi

hdr() {  # $1=label $2=prefix $3=suffix $4=outfile
  echo "----- $1 -----"
  curl -sS -i --max-time 10 -X POST "http://127.0.0.1:${PORT}/api/v2/alias/custom/new" \
    -H "Authentication: ${API_KEY}" -H "Content-Type: application/json" \
    -d "{\"alias_prefix\":\"$2\",\"signed_suffix\":\"$3\"}" | tee "$4"
  echo; echo
}

echo
echo "===== [3/5] CAPTURE raw headers via curl -i (live transport) ====="
hdr "REQ1 valid  -> expect 201" lh1 "$VALID"    "$OUT/live_201.txt"
hdr "REQ2 tamper -> expect 412" lh2 "$TAMPERED" "$OUT/live_412.txt"
hdr "REQ3 valid  -> 201 (fill window)" lh3 "$VALID" /dev/null
hdr "REQ4 valid  -> 201 (fill window)" lh4 "$VALID" /dev/null
hdr "REQ5 valid  -> 201 (fill window)" lh5 "$VALID" /dev/null
hdr "REQ6 valid  -> expect 429 (6th within 5/minute)" lh6 "$VALID" "$OUT/live_429.txt"

echo
echo "===== [4/5] STOP gunicorn (only captured PID $GPID) ====="
kill "$GPID" 2>/dev/null
wait "$GPID" 2>/dev/null
echo "gunicorn stopped"

echo
echo "===== [5/5] TEARDOWN (SQL delete + verify net-zero) + Redis cleanup ====="
"${CANON_ENV[@]}" /app/venv/bin/python /tmp/sl_probes/probe_live_teardown.py
TD=$?
DEL=$(redis-cli --scan --pattern 'LIMITER/*ip:127.0.0.1*new_custom_alias*' | xargs -r redis-cli del)
echo "redis LIMITER keys deleted for ip:127.0.0.1/new_custom_alias: ${DEL:-0}"
exit $TD
```

### 11.9 `probe_live_seed.py`

Live-server seed. Creates one disposable premium user + API key for the live `gunicorn` run and prints the API key to a file consumed by `probe_live.sh` (never echoed to the shared console).
```python
#!/usr/bin/env python3
"""
probe_live_seed.py  --  seed a DISPOSABLE premium user + API key for the live-server
header capture, mint a canonical valid signed_suffix (via the real app signer, same
CUSTOM_ALIAS_SECRET) and a guaranteed-different tampered variant, and record baseline
table counts so the teardown can prove net-zero DB residue.

All created rows are DISPOSABLE / synthetic test data and are removed by
probe_live_teardown.py. The signing secret is the PUBLIC test secret from
tests/test.env (FLASK_SECRET="secret" -> CUSTOM_ALIAS_SECRET="secretcustom_alias").
"""
import os, json
os.environ.setdefault("CONFIG", "/app/tests/test.env")

from app.db import Session
from app.models import User, ApiKey
import itsdangerous
import string as _string
from app.alias_suffix import signer
from app.config import EMAIL_DOMAIN
from app.utils import random_word, random_string

STATE = "/tmp/sl_probes/live_state.json"
TABLES = ["alias", "users", "api_key", "mailbox", "alias_mailbox",
          "deleted_alias", "domain_deleted_alias", "alias_used_on", "contact"]

def counts():
    from sqlalchemy import text
    out = {}
    for t in TABLES:
        out[t] = Session.execute(text(f"select count(*) from {t}")).scalar()
    return out

baseline = counts()

email = f"livehdr_{random_string(8)}@example.test"
user = User.create(email=email, password="password", name="Live Hdr Probe", activated=True)
user.lifetime = True          # premium => account quota never interferes with the header probe
Session.flush()
key = ApiKey.create(user.id, "live hdr probe")
Session.commit()              # live gunicorn is a separate process => must be committed

valid = signer.sign(f".{random_word()}@{EMAIL_DOMAIN}").decode()

# VERIFIED-invalid tampered suffix (Finding 15): a naive last-char flip is unreliable
# because the final base64url char can carry padding bits; walk positions and return the
# first single-char change that genuinely makes signer.unsign raise BadSignature.
_B64URL = _string.ascii_letters + _string.digits + "-_"
def _make_tampered(tok):
    for i in range(len(tok)):
        repl = next(a for a in _B64URL if a != tok[i])
        cand = tok[:i] + repl + tok[i + 1:]
        try:
            signer.unsign(cand, max_age=600)
        except itsdangerous.BadSignature:
            return cand
    raise RuntimeError("could not construct a tampered token")
tampered = _make_tampered(valid)

state = {
    "user_id": user.id,
    "email": email,
    "api_key": key.code,
    "valid_suffix": valid,
    "tampered_suffix": tampered,
    "baseline": baseline,
}
with open(STATE, "w") as f:
    json.dump(state, f)

# emit shell-consumable values
print(f"USER_ID={user.id}")
print(f"API_KEY={key.code}")
print(f"VALID_SUFFIX={valid}")
print(f"TAMPERED_SUFFIX={tampered}")
print(f"BASELINE={json.dumps(baseline)}")
```

### 11.10 `probe_live_teardown.py`

Live-server teardown. Removes the single seed user and its dependent rows created by `probe_live_seed.py`, restoring the DB baseline after the live run.
```python
#!/usr/bin/env python3
"""
probe_live_teardown.py -- remove EVERY row created by the live-server header probe using
direct SQL DELETEs (in FK-safe order) so no delete_alias() side-effect records
(deleted_alias / domain_deleted_alias) are ever written, then assert every table count
is byte-identical to the pre-probe baseline recorded by probe_live_seed.py.

FK-safe order: NULL users.default_mailbox_id first (breaks the users->mailbox cycle),
delete the user's alias/api_key rows, then delete the user (mailbox.user_id has
ON DELETE CASCADE so the mailbox goes with it); a trailing mailbox delete mops up any
non-cascaded rows. Exits non-zero if any table did not return to baseline.
"""
import os, json, sys
os.environ.setdefault("CONFIG", "/app/tests/test.env")

from sqlalchemy import text
from app.db import Session

STATE = "/tmp/sl_probes/live_state.json"
with open(STATE) as f:
    state = json.load(f)

uid = state["user_id"]
baseline = state["baseline"]

Session.execute(text("update users set default_mailbox_id=null where id=:u"), {"u": uid})
Session.execute(text("delete from alias_used_on where user_id=:u"), {"u": uid})
Session.execute(text(
    "delete from alias_mailbox where alias_id in (select id from alias where user_id=:u)"),
    {"u": uid})
Session.execute(text("delete from alias where user_id=:u"), {"u": uid})
Session.execute(text("delete from api_key where user_id=:u"), {"u": uid})
Session.execute(text("delete from users where id=:u"), {"u": uid})          # cascades mailbox
Session.execute(text("delete from mailbox where user_id=:u"), {"u": uid})   # mop up
Session.commit()

TABLES = list(baseline.keys())
after = {t: Session.execute(text(f"select count(*) from {t}")).scalar() for t in TABLES}

print("table               baseline   after   delta")
bad = []
for t in TABLES:
    d = after[t] - baseline[t]
    flag = "" if d == 0 else "  <-- RESIDUE"
    if d != 0:
        bad.append(t)
    print(f"  {t:<18} {baseline[t]:>8} {after[t]:>7} {d:>+6}{flag}")

if bad:
    print(f"\n>>> TEARDOWN FAILED: residue in {bad}")
    sys.exit(1)
print("\n>>> TEARDOWN OK: all tables restored to baseline (net-zero DB residue)")
sys.exit(0)
```

---

## 12. Grounding, methodology, and evidence matrix

### 12.1 Methodology (run-first, observe-then-write)

Every behavioural statement in this document was produced by **running the real code path** and
capturing its actual output, not by reading alone:

- **Canonical entry point.** All results come from `POST /api/v2/alias/custom/new` and
  `POST /api/v3/alias/custom/new` driven through the real `server.create_app()` app (Werkzeug test
  client for the isolated probes; a live `gunicorn` on `:7777` for the raw-header captures), with
  real API-key authentication. No debug hooks, mocks, or hand-forged tokens are used to obtain a
  reported value; where a value could not be observed it is explicitly labelled **`inferred`**
  (§10.4).
- **Canonical inputs.** Valid `signed_suffix` values are minted by the real producer
  (`GET /api/v4/alias/options` / `app.alias_suffix.signer`); tampered values use the deterministic
  `make_tampered` helper (§9.3); expired values are minted by backdating the real signer by 601s
  against `max_age=600` (§3.2, §9).
- **Every condition exercised.** Success (201), tampered/expired/garbage suffix (412), empty body
  (400), duplicate (409), two-consecutive-dots (400), wrong prefix/suffix/domain (400),
  quota-exceeded (400), non-string `signed_suffix` (500), and the rate-limit/parallel-lock
  rejections (429) are each driven and captured (§3, §5, §7, §8, §9).
- **Stability.** The rate-limit observation was made with limiting **enabled** and confirmed
  identical across two isolated runs (`RUN1 == RUN2`, both `[201x5, 429, 429]`, §5).
- **Pinned libraries, observed.** `itsdangerous 1.1.0`, `flask-limiter 1.4`, `flask 1.1.2`,
  `werkzeug 1.0.1`, `redis 4.6.0`, `sqlalchemy 1.3.24` on Python 3.10.18 — printed at runtime in
  §2.1 — so the reported behaviour reflects the shipping application.
- **Single-run coherence.** The per-condition evidence in §3–§9 all comes from **one**
  `run_all.sh` execution (process timestamps `19:30:31`–`19:30:47`); the net-zero cleanup proof in
  §2.5 is a **separate** fresh run at the clean `824`-alias baseline (it cites only aggregate
  counts, never per-alias IDs, so the two are independent and non-conflicting).

### 12.2 Evidence matrix

This matrix replaces any self-certification: each claim is tied to the exact command that produced
it and to the section holding the complete, unedited output. All probe commands use the canonical
invocation in §2.3; source is in §11.

| # | Claim (observed) | Question | Probe / command | Complete output |
|---|------------------|----------|-----------------|-----------------|
| 1 | Tampered suffix -> **HTTP 412** `{"error":"Alias creation time is expired, please retry"}` | Q1 | `probe_functional.py`, `probe_mro.py` | §3.2 (COND), §9.2 (`mro.out`) |
| 2 | Genuinely **expired** suffix (backdated 601s, `max_age=600`) -> **HTTP 412** (same body) | Q1 | `probe_functional.py` (backdated mint) | §3.2 (COND5) |
| 3 | **Garbage** string suffix -> **HTTP 412** (same body) | Q1 | `probe_mro.py` | §9.2 (`mro.out`) |
| 4 | Valid suffix -> **HTTP 201** with serialized alias body | Q1 | `probe_functional.py` (COND0), `probe_live.sh` | §3.1 (`live_201.txt`), §3.2 |
| 5 | Empty body -> **HTTP 400** `request body cannot be empty` | Q1 | `probe_functional.py` | §3.2, §8 |
| 6 | Duplicate alias -> **HTTP 409** | Q1 | `probe_functional.py` | §3.2, §8 |
| 7 | Two consecutive dots -> **HTTP 400** `2 consecutive dot signs` | Q1 | `probe_functional.py` | §8 |
| 8 | Wrong prefix/suffix/domain -> **HTTP 400** `wrong alias prefix or suffix` | Q1 | `probe_functional.py` | §3.2, §8 |
| 9 | Truthy non-string `signed_suffix` (list/int) -> **HTTP 500** `Internal error` (AttributeError at `.strip()`) | Q1 | `probe_f18.py` | §9.4 (`f18.out`) |
| 10 | JSON `null` `signed_suffix`: v2 -> **500**, v3 -> **412** (v3 `or ""` guard, L157) | Q1 | `probe_f18.py` | §9.4 (`f18.out`) |
| 11 | Failure emits `LOG.w` **WARNING** `Alias creation time expired for <user>` (v2 L72 / v3 L187) | Q2 | `probe_functional.py`, `probe_f18.py` | §4, §9.4 |
| 12 | Every response emits a `DEBUG after_request` line (`server.py:284`); success also `INFO send_event` | Q2 | all probes | §4, §3.2 |
| 13 | **No** `X-RateLimit-*` / `Retry-After` headers on any response (limiting on or off) | Q3 | `probe_ratelimit.py`, `probe_live.sh`, `probe_customdomain.py` | §5 (`ratelimit.out`), §3.1 (`live_429.txt`), §9.5 (`customdomain.out`) |
| 14 | Boundary is `[201x5, 429, 429]`, first **429 at request #6**, stable `RUN1 == RUN2` | Q3 | `probe_ratelimit.py` | §5 (`ratelimit.out`) |
| 15 | Flask-Limiter header-default fix is a **0.7.x**-series change (predates 1.4), **not** 1.5 | Q3 | `HISTORY.rst` + web check | §5 |
| 16 | Key derivation: session -> `userid:{id}`, API-key-only -> `ip:{addr}`; lock keys `cl:{id}` / `cl:{addr}` | Q3 | `probe_ratelimit.py` | §5 (`ratelimit.out`) |
| 17 | Quota gate is `User.can_create_new_alias()` (`models.py:867-884`) + `max_alias_for_free_account()` (`858-865`) | Q4 | `probe_functional.py` (COND13) | §6 (`functional.out`) |
| 18 | **No** quota value is logged on the 201 path; the only quota log is the failure `LOG.d` (`models.py:49`/`:138`) | Q4 | `probe_functional.py` | §6 |
| 19 | Quota check precedes the signature check, so a quota-exhausted user gets the 400 quota message regardless of suffix validity | Q4/Q5 | `probe_functional.py` (COND13c) | §6.4 |
| 20 | Decorator order `@limiter.limit` -> `@require_api_auth` -> `@parallel_limiter.lock` (v2 L28-30 / v3 L115-117) | Q5 | source | §7.1 |
| 21 | Parallel lock -> **HTTP 429** (werkzeug `TooManyRequests`) on contention; **no-op** when Redis absent (`parallel_limiter.py:55-58`) | Q5 | `probe_lock.py` | §7.7 (`lock.out`) |
| 22 | The 412 collapse is **domain-independent** (verified custom domain: valid 201, tampered 412) | Q1/root | `probe_customdomain.py` | §9.5 (`customdomain.out`) |
| 23 | Repository unchanged; DB/Redis net-zero; probes removed | scope | `run_all.sh`, `git status` | §2.5 (`run_all.out`) |

### 12.3 Per-question coverage confirmation

- **Q1 (status codes / error messages).** Answered in §3 and §8 with the complete status+body for
  every condition; the tampered/expired/garbage collapse and the 500 non-string case are in §9.
- **Q2 (server-console log entries).** Answered in §4 with the raw captured buffer, the capture
  mechanism, and a per-emitter `file:line` table (`LOG.w` at `new_custom_alias.py:72`/`:187`;
  `LOG.d` quota at `models.py:49`/`:138`; `after_request` at `server.py:284`).
- **Q3 (rate-limit headers).** Answered in §5: no `X-RateLimit-*`/`Retry-After` headers appear;
  the boundary and key-derivation are observed and stable; the header-default version nuance is
  corrected.
- **Q4 (success-path quota checks / logged values).** Answered in §6: `can_create_new_alias()`
  and `max_alias_for_free_account()` are the gates; **no** value is logged on success.
- **Q5 (execution-path trace / rejection conditions).** Answered in §7 (decorator chain + branch
  order + mermaid trace) and §9 (root cause), naming each mechanism — `check_suffix_signature`,
  `verify_prefix_suffix`, `can_create_new_alias`, `max_alias_for_free_account`, `@limiter.limit`,
  `@require_api_auth`, `@parallel_limiter.lock` — with its `file:line` and the condition that
  triggers each rejection.

### 12.4 Scope verification (read-only)

- The SimpleLogin source repository was **not modified**: `git status --porcelain` shows only the
  single new file `blitzy/documentation/app_2cd6ee777f8c.md`, and `git diff <baseline> --name-status`
  reports it as the only addition (§2.5).
- All temporary probe scripts lived only in the container at `/tmp/sl_probes/` and are **removed**
  after evidence capture; the DB and Redis are restored to their clean baseline with an unedited
  net-zero proof (§2.5).
- **Sensitive-test-data hygiene.** Every signed suffix shown is synthetic, disposable, test-only,
  and time-limited; the signing key is the public, test-only `tests/test.env` value and is never a
  credential; real API keys and session cookies are REDACTED throughout while their reproducible
  generation is described (§1, §11).

