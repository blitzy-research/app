# How SimpleLogin Creates a New Email Alias — End‑to‑End, Runtime‑Verified

> **Scope of this document.** This is a read‑only investigation answer. It explains, *from observed runtime behavior* of the running SimpleLogin Flask application, exactly what happens when a user creates a new email alias: the request the frontend sends, the backend response, the database changes, the background/follow‑up work, and the failure behavior. Every factual claim is backed by either **raw captured output** (shown before any summary) or a **`file:line`** reference into the source. Statements that could only be established by reading (not running) are explicitly **labeled `[inferred]`**.
>
> All observations were made by driving the **real HTTP entry points** (the web form `POST /dashboard/` and the JSON API `POST /api/alias/random/new`) against a live `gunicorn` server, so CSRF, `@login_required`, the API‑key decorator, the rate limiters, the parallel lock, and the commit boundaries were all in force. No internal helper was called directly to fabricate a result. No repository source file was modified; the only persistent artifact is this document.

---

## 1. TL;DR — Direct Answer

**Both entry points converge on the same persistence core, so the database effects are identical; only the transport and the response differ.** The web form handler `dashboard.index` and the JSON API `new_random_alias` both call `Alias.create_new_random` (`app/models.py:L1721`) → `Alias.create` (`app/models.py:L1628-L1692`).

- **Frontend request (O4).** The dashboard "Random Alias" button submits an **`application/x-www-form-urlencoded` `POST` to `/dashboard/`** carrying two fields: `form-name=create-random-email` and `csrf_token=<…>` (plus an *optional* `generator_scheme` when the "By Random Words" / "By UUID" dropdown items are used). Evidence: `templates/dashboard/index.html:L50-L58` (main button), `L68-L86` (dropdown variants); captured request below (§3).

- **Backend response (O5).**
  - **Web → HTTP `302 FOUND`**, `Location: http://localhost:7777/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=` (`app/dashboard/views/index.py:L113-L121`).
  - **API → HTTP `201 CREATED`** with a JSON body of **17 keys** = the 16 keys from `serialize_alias_info_v2` (`app/api/serializer.py:L55-L93`) plus a top‑level `alias` string (`app/api/views/new_random_alias.py:L114-L116`).

- **Database changes (O6).** In the default, seeded, single‑mailbox configuration a successful creation **touches exactly THREE tables**, confirmed stable across repeated runs:
  1. `alias` — one **INSERT** (`app/models.py:L1660`).
  2. `daily_metric` — the day's row is **UPDATE**d (`nb_alias += 1`) on every creation after the first of the calendar day, or **INSERT**ed on the first creation of a new day (`app/models.py:L1661`, `DailyMetric.get_or_create_today_metric` `app/models.py:L3280`).
  3. `alias_audit_log` — one **INSERT** with `action='create'`, `message='New alias created'` (`app/models.py:L1688-L1690`, `app/alias_audit_log_utils.py:L18-L32`).
  There are **no writes** to `sync_event`, `alias_mailbox`, `users`, or `alias_used_on` in this default flow (all observed at zero‑delta; §5).

- **Background tasks / follow‑up work (O7).** **None** in default configuration. Alias creation emits exactly one creation log line (`app/dashboard/views/index.py:L110`, web path only) and then event dispatch **short‑circuits** on the *second* early return in `EventDispatcher.send_event` because `EVENT_WEBHOOK` is unset — logging `Not sending events because webhook is not configured and allowed to be empty` (`app/events/event_dispatcher.py:L61-L65`). Consequently **no `sync_event` row is written and no `NOTIFY` is issued**, so `event_listener.py` and `job_runner.py` do no alias‑creation follow‑up.

- **Failure behavior (O8).** Each failure surfaces distinctly and, on refusal, writes **nothing** to the database:
  - **Free‑plan cap** (`MAX_NB_EMAIL_FREE_PLAN=5`): API → **`400`** JSON error; web → **`302`** + flash *"You need to upgrade your plan to create new alias."*
  - **Trashed‑alias reuse**: `Alias.create` raises **`AliasInTrashError`** (`app/models.py:L1647-L1652`); on the random endpoint's hostname branch it is **caught** and the endpoint falls back to a random alias (still `201`); on the custom‑alias endpoint a trashed email is pre‑checked and returns **`409`**.
  - **Invalid CSRF** (web): flash *"Invalid request"* + **`302`** back to `request.url`.
  - **Invalid API `mode`**: **`400`** `{"error":"<mode> must be either word or uuid"}`.
  - **Rate limiting**: **`429`** `{"error":"Rate limit exceeded"}` once the flask‑limiter bucket (`ALIAS_LIMIT="100/day;50/hour;5/minute"`) is exceeded.

> **Critical nuance carried throughout.** The seeded user `john@wick.com` is **premium** (active subscription), so the free‑plan cap does *not* apply to him; the cap was therefore reproduced with a **separately created free user** (`freebie@sl.local`), clearly labeled wherever used (§7.1, §2.4).

---

## 2. Environment & Methodology

### 2.1 Platform and how the app was run

The application was already stood up (canonical dev runtime) inside the attached Docker container and observed live. The exact serving process and its version:

```
$ docker exec sl-app bash -lc 'ps -o pid,cmd -C gunicorn | head'
  PID CMD
  808 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30
  809 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30
  810 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30
```

- **Server:** `gunicorn/20.0.4`, WSGI entry `wsgi:app`, bound `0.0.0.0:7777`, **2 workers** (master PID 808, workers 809 & 810). The two‑worker fact matters for the in‑memory rate limiter (§7.5).
- **Why `gunicorn` and not `python3 server.py`:** `server.py:L588` binds `127.0.0.1` only (`app.run(debug=True, ..., port=7777)`), which would not be reachable from the host; the container therefore serves host‑accessible via `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2`. The Flask app object is the same factory `create_app` (`server.py:L139`); `wsgi.py` imports it.
- **Health check** (confirms the app is live and redirects anonymous users to login):

```
$ curl -sS -i http://localhost:7777/ | head -5
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Content-Type: text/html; charset=utf-8
Content-Length: 219
Location: http://localhost:7777/auth/login
```

### 2.2 Datastore, cache, and configuration

- **PostgreSQL 13** in container `sl-db`. The app reaches it over the Docker network alias `sl-db:5432`; the host publishes it on `55432`. **DB port reconciliation:** the app's `.env` uses `DB_URI=postgresql://myuser:mypassword@sl-db:5432/simplelogin` (the in‑network port `5432`), which is the port actually used; the host‑published `55432` is only for host‑side access. I ran all SQL through the container, so the effective port was **`sl-db:5432`**.
- **Redis** container `sl-redis` exists, but **`MEM_STORE_URI` is unset** in `.env`, which is the canonical dev setting. Consequence (runtime‑verified below): the Redis‑backed limiters are **not initialized**, so the flask‑limiter storage falls back to **in‑memory**, and the in‑`Alias.create` token bucket and the parallel lock become **no‑ops** (§7.5).
- **Key config values, read at runtime from inside the app process:**

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
from app import config
for k in [\"MEM_STORE_URI\",\"DISABLE_RATE_LIMIT\",\"EVENT_WEBHOOK\",\"EVENT_WEBHOOK_DISABLE\",
          \"MAX_NB_EMAIL_FREE_PLAN\",\"ALIAS_LIMIT\",\"ALIAS_CREATE_RATE_LIMIT_FREE\",
          \"ALIAS_CREATE_RATE_LIMIT_PAID\",\"FIRST_ALIAS_DOMAIN\"]:
    print(k, \"=\", getattr(config,k))
"'
MEM_STORE_URI = None
DISABLE_RATE_LIMIT = False
EVENT_WEBHOOK = None
EVENT_WEBHOOK_DISABLE = False
MAX_NB_EMAIL_FREE_PLAN = 5
ALIAS_LIMIT = 100/day;50/hour;5/minute
ALIAS_CREATE_RATE_LIMIT_FREE = [(10, 900), (50, 3600)]
ALIAS_CREATE_RATE_LIMIT_PAID = [(50, 900), (200, 3600)]
FIRST_ALIAS_DOMAIN = sl.local
```

These correspond to `app/config.py`: `MAX_NB_EMAIL_FREE_PLAN` default 5 (`L124`), `ALIAS_LIMIT` (`L448`), `ALIAS_CREATE_RATE_LIMIT_FREE/PAID` (`L554-L558`), `MEM_STORE_URI` (`L568`), `DISABLE_RATE_LIMIT` (`L602`), `EVENT_WEBHOOK` default `None` (`L612`), `EVENT_WEBHOOK_DISABLE` (`L616`). Other bounding defaults from `.env`: `URL=http://localhost:7777`, `EMAIL_DOMAIN=sl.local`, `NOT_SEND_EMAIL=true`.

### 2.3 Logging

`app/log.py` configures a single logger named `SL` at `DEBUG` level writing to stdout, captured by gunicorn to `/tmp/gunicorn.log` inside `sl-app`. `LOG.d`=debug, `LOG.i`=info, `LOG.w`=warning, `LOG.e`=exception. Every HTTP request is logged by `server.py:284 after_request()` in the form `<ip> <METHOD> <path> <args> <status>, takes <t>`. All log excerpts below are sliced from `/tmp/gunicorn.log`.

### 2.4 Users, credentials, and the premium nuance

The seeded users were inspected read‑only through the running app process:

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
from app.models import User
j=User.get(1)
print(\"john email=\",j.email,\"| is_premium=\",j.is_premium(),
      \"| lifetime_or_active_subscription=\",j.lifetime_or_active_subscription(),
      \"| can_create_new_alias=\",j.can_create_new_alias())
print(\"john active_subscription=\", j.get_active_subscription())
"'
john email= john@wick.com | is_premium= True | lifetime_or_active_subscription= True | can_create_new_alias= True
john active_subscription= <Subscription PlanEnum.monthly 2026-07-18>
```

- `john@wick.com` / `password` is **premium** — an active monthly `Subscription` seeded at `app/fake_data.py:L106-L119`. Because `can_create_new_alias()` returns `True` immediately for a user with `lifetime_or_active_subscription()` (`app/models.py:L867`, `L746`), **the free‑plan cap is never reached for john.** He was used for all happy‑path and non‑cap tests.
- To exercise the **free‑plan cap** (O8), a **free user was created at runtime** (labeled controlled setup, not a source change): `freebie@sl.local` (id 3) with API key `freecode`, pre‑filled to the 5‑alias cap. Its free status was confirmed live: `can_create_new_alias() == False` (§7.1).
- **API authentication** uses the `Authentication` HTTP header carrying an API‑key `code` (`app/api/base.py:L16-L34`). John's seeded keys are `code` (Chrome) and `codeFF` (Firefox) (`app/fake_data.py:L121-L125`). API calls therefore need **no session cookie**.

### 2.5 The read‑only row‑count probe

To measure database deltas I used a temporary, **SELECT‑only** probe (kept under `/tmp/scratch_blitzy_qna/probe.sh`, outside the repo tree, deleted at cleanup). It runs a single atomic `UNION ALL` count over every candidate table:

```
$ cat /tmp/scratch_blitzy_qna/probe.sh
#!/usr/bin/env bash
docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tA -c "
  select '\''alias='\''         || count(*) from alias            union all
  select '\''alias_audit_log='\''|| count(*) from alias_audit_log  union all
  select '\''alias_mailbox='\''  || count(*) from alias_mailbox    union all
  select '\''alias_used_on='\''  || count(*) from alias_used_on    union all
  select '\''daily_metric='\''   || count(*) from daily_metric     union all
  select '\''job='\''            || count(*) from job              union all
  select '\''sync_event='\''     || count(*) from sync_event       union all
  select '\''users='\''          || count(*) from users;"'

$ bash /tmp/scratch_blitzy_qna/probe.sh    # baseline, before any test creation
alias=13
alias_audit_log=13
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2
```

The candidate table set covers the alias‑creation core (`alias`, `daily_metric`, `alias_audit_log`), the entities that *could* be created as side effects (`alias_mailbox`, `alias_used_on`, `users`), and the event/async surfaces (`sync_event`, `job`). Snapshots were taken **before and after** each creation, and creations were repeated to confirm magnitude stability.

> **Note on the baseline `daily_metric=1`.** Because the seed process created a `daily_metric` row for the current day, same‑day creations **UPDATE** that row rather than inserting a new one. The first‑of‑day **INSERT** sub‑case is demonstrated separately in §5.3 by deleting the current day's `daily_metric` row at runtime (a labeled, runtime‑only DB action) and observing the re‑INSERT.

### 2.6 Repository integrity & cleanup

All observation scripts live under `/tmp/scratch_blitzy_qna` (host) and `/tmp` (container) — never inside the repository. They are deleted at completion, and the leftover `blitzy/screenshots/` from a prior session is removed, leaving only this document. Repository‑unchanged verification (`git status --porcelain` showing only the new `blitzy/documentation/` path, and `git diff` empty for tracked files) is performed at the end and reported in §8.4.

> **Runtime DB mutations disclosure.** The observations mutated the *running database only* (new aliases 14–31, a runtime‑created free user id 3, one deleted/trashed alias, an incremented API‑key usage counter, and a temporarily deleted `daily_metric` row). These are ephemeral runtime state, **not** repository changes; no tracked file was touched.

---

## 3. O4 — What request the frontend sends to the backend

### 3.1 Raw captured request

The exact request the browser issues when the "Random Alias" button is clicked, captured by driving the same request `curl` issues against the running server as authenticated `john` (trace shows the request line, headers, and body verbatim):

```
$ curl -sS --trace-ascii /tmp/scratch_blitzy_qna/req1.trace -b john.cookies -c john.cookies \
       -o /dev/null -D /tmp/scratch_blitzy_qna/resp1.hdr \
       -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=IjdhMzc5MGUxZDQ4M2U3ODQwOThiM2ZiYzdkMGEyMTFlNWJjZDIyOGUi.ak3SjQ.AC4dw9l4dQJnIV49jc1Mj-VkaMw'

# ---- request as sent on the wire (from req1.trace) ----
POST /dashboard/ HTTP/1.1
Host: localhost:7777
Cookie: slapp=<authenticated session cookie for john@wick.com>
Content-Type: application/x-www-form-urlencoded
Content-Length: 132

form-name=create-random-email&csrf_token=IjdhMzc5MGUxZDQ4M2U3ODQwOThiM2ZiYzdkMGEyMTFlNWJjZDIyOGUi.ak3SjQ.AC4dw9l4dQJnIV49jc1Mj-VkaMw
```

### 3.2 What this shows, and the evidence

- **Method + URL:** `POST /dashboard/` — **not** `POST /`. The form (`templates/dashboard/index.html:L50` `<form method="post">`) has **no `action` attribute**, so it submits to the current document URL, which is `/dashboard/` because the dashboard blueprint is mounted with `url_prefix="/dashboard"` (`app/dashboard/base.py`). The root `/` view merely redirects authenticated users to `dashboard.index` (`server.py:L251-L255`), confirmed live: `GET /` (authenticated) → `302 Location /dashboard/`. **This corrects the plain "`POST /`" reading — the real target is `/dashboard/`.**
- **Content type:** `application/x-www-form-urlencoded` (a classic HTML form submit; no JSON, no multipart).
- **Body fields (exactly two required):**
  - `form-name=create-random-email` — selects the random‑alias branch inside the multiplexed dashboard handler (`app/dashboard/views/index.py`; the `create-random-email` branch guards on this value).
  - `csrf_token=<91‑char token>` — the Flask‑WTF CSRF token rendered by `{{ csrf_form.csrf_token }}` in the template.
- **Optional third field `generator_scheme`:** the main "Random Alias" button does **not** send it (template `L50-L58`); the dropdown items add it — **"By Random Words" → `generator_scheme=1`** (`AliasGeneratorEnum.word.value`, template `L68-L76`) and **"By UUID" → `generator_scheme=2`** (`AliasGeneratorEnum.uuid.value`, template `L78-L86`). Runtime confirmation of the values: omitting it (or `=1`) produced word‑style aliases (`behave_wander222@sl.local`, `logier_gantry894@sl.local`); `generator_scheme=2` produced a uuid‑style alias (`5dd3ea1d-5506-4230-a536-76ee8eecff00@sl.local`).

> The enum values were confirmed at runtime by rendering the dashboard and reading the hidden inputs; `word=1`, `uuid=2`.

---

## 4. O5 — The backend response (status codes, payloads, metadata)

### 4.1 Web flow → HTTP 302 redirect

Raw response for a successful web creation (same request as §3.1):

```
$ sed -n '1,10p' /tmp/scratch_blitzy_qna/resp1.hdr
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 04:31:30 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 339
Location: http://localhost:7777/dashboard/?highlight_alias_id=14&query=&sort=&filter=
Vary: Cookie
Set-Cookie: slapp=<new session cookie carrying the success flash>; HttpOnly; Path=/; SameSite=Lax
```

- **Status: `302 FOUND`.** The body is the standard 339‑byte redirect HTML pointing at the `Location`.
- **`Location: /dashboard/?highlight_alias_id=<id>&query=&sort=&filter=`.** The newly created alias's id is passed as `highlight_alias_id` so the redirected page can highlight it. Evidence: `app/dashboard/views/index.py:L113-L121` (`redirect(url_for("dashboard.index", highlight_alias_id=alias.id, query=…, sort=…, filter=…))`). The id in `Location` is the only part that varies per creation (observed `14`, `15`, `16`, `17` across runs).
- **Metadata:** a fresh `Set-Cookie: slapp=…` carries the success flash *"Alias &lt;email&gt; has been created"* (`app/dashboard/views/index.py:L111`), rendered on the next page load.

### 4.2 API flow → HTTP 201 with serialized JSON

Raw response for a successful API creation (no cookie; API‑key header only):

```
$ curl -sS -i -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: code'
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 04:35:57 GMT
Content-Type: application/json
Content-Length: 406
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<fresh anonymous session>; HttpOnly; Path=/; SameSite=Lax

{"alias":"pecked_breezy873@sl.local","creation_date":"2026-07-08 04:35:57+00:00","creation_timestamp":1783485357,"disable_pgp":false,"email":"pecked_breezy873@sl.local","enabled":true,"id":18,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
```

- **Status: `201 CREATED`.** Metadata: `Content-Type: application/json`, `Access-Control-Allow-Origin: *`. The `Set-Cookie` is an incidental fresh anonymous session (API auth is by header, not cookie).
- **Keys are alphabetically sorted** because Flask's `JSON_SORT_KEYS` defaults to `True`.

### 4.3 Every JSON key enumerated → the source line that produces it

The body has **17 keys**: 16 from `serialize_alias_info_v2` (`app/api/serializer.py:L55-L93`) plus the top‑level `alias` added by the endpoint (`app/api/views/new_random_alias.py:L115`, `jsonify(alias=alias.email, **serialize_alias_info_v2(...))`).

| JSON key | Observed value (id 18) | Source line |
|----------|------------------------|-------------|
| `alias` | `"pecked_breezy873@sl.local"` | `new_random_alias.py:L115` (top‑level, = the alias email) |
| `id` | `18` | `serializer.py:L58` |
| `email` | `"pecked_breezy873@sl.local"` | `serializer.py:L59` |
| `creation_date` | `"2026-07-08 04:35:57+00:00"` | `serializer.py:L60` (`created_at.format()`) |
| `creation_timestamp` | `1783485357` | `serializer.py:L61` (`created_at.timestamp`) |
| `enabled` | `true` | `serializer.py:L62` |
| `note` | `null` | `serializer.py:L63` |
| `name` | `null` | `serializer.py:L64` |
| `nb_forward` | `0` | `serializer.py:L66` |
| `nb_block` | `0` | `serializer.py:L67` (from `alias_info.nb_blocked`) |
| `nb_reply` | `0` | `serializer.py:L68` |
| `mailbox` | `{"email":"john@wick.com","id":1}` | `serializer.py:L70` |
| `mailboxes` | `[{"email":"john@wick.com","id":1}]` | `serializer.py:L71-L74` |
| `support_pgp` | `false` | `serializer.py:L75` (`mailbox_support_pgp()`) |
| `disable_pgp` | `false` | `serializer.py:L76` |
| `latest_activity` | `null` | `serializer.py:L77` (populated only if a `latest_email_log` exists, `L80-L92`) |
| `pinned` | `false` | `serializer.py:L78` |

`latest_activity` is `null` for a freshly created alias because there is no email log yet — verified: the field is populated only inside the `if alias_info.latest_email_log:` block at `serializer.py:L80-L92`, which is not reached for a new alias. (The `serialize_alias_info_v2` dict literal spans `serializer.py:L56-L79`; the comment `# Alias field` sits at `L57`, so the first key `id` is at `L58`.)

### 4.4 Direct comparison — the responses differ, the effects do not

The web and API responses differ **only** in transport and shape (`302` redirect vs `201` + JSON). The database work behind them is identical because both call the same `Alias.create` (proved by the identical 3‑table delta in §5.4). This is the central "no difference where it matters" result.


---

## 5. O6 — Database changes (records inserted/updated, how many tables, related entities)

**Direct answer:** a successful creation in the default single‑mailbox configuration **touches exactly three tables** — `alias` (INSERT), `daily_metric` (INSERT on the first alias of a calendar day, otherwise UPDATE of `nb_alias`), and `alias_audit_log` (INSERT). No related entities (`alias_mailbox`, `alias_used_on`, `users`, `sync_event`) are written on this path. The count is stable across runs.

### 5.1 Web creation, run 1 (same day → `daily_metric` UPDATE)

```
$ bash /tmp/scratch_blitzy_qna/probe.sh   # BEFORE run 1
alias=13
alias_audit_log=13
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2

$ curl -sS -o /dev/null -D - -b john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid token>' | grep -i '^HTTP\|^Location'
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=14&query=&sort=&filter=

$ bash /tmp/scratch_blitzy_qna/probe.sh   # AFTER run 1
alias=14
alias_audit_log=14
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2
```

**Delta run 1:** `alias` **+1**, `alias_audit_log` **+1**, `daily_metric` row‑count **+0** (the existing day row was UPDATED — `nb_alias` went `13 → 14`, shown in §5.3), everything else **+0**.

### 5.2 Web creation, run 2 — identical input, magnitude stability

```
$ bash /tmp/scratch_blitzy_qna/probe.sh   # BEFORE run 2
alias=14
alias_audit_log=14
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2

$ curl -sS -o /dev/null -D - -b john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid token>' | grep -i '^HTTP\|^Location'
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=15&query=&sort=&filter=

$ bash /tmp/scratch_blitzy_qna/probe.sh   # AFTER run 2
alias=15
alias_audit_log=15
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2
```

**Delta run 2:** identical shape — `alias` **+1**, `alias_audit_log` **+1**, `daily_metric` row‑count **+0** (UPDATE, `nb_alias 14 → 15`), all else **+0**. **The number of tables touched is stable at THREE across both runs.** (It remained 3 on every subsequent creation in §5.4 and §7 as well.)

### 5.3 The `daily_metric` INSERT‑vs‑UPDATE sub‑cases

`Alias.create` does `DailyMetric.get_or_create_today_metric().nb_alias += 1` (`app/models.py:L1661`). `get_or_create_today_metric` (`app/models.py:L3280`) fetches the row for today's date or **creates** it if absent. So:

- **Same day (row already exists):** the row is **UPDATE**d; `daily_metric` row‑count does not change. Proof — `nb_alias` incremented across the two runs above:

```
$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc \
  "select id, date, nb_alias from daily_metric where date = current_date;"'
1|2026-07-08|15          # after run 2 — same single row, nb_alias climbed 13 -> 14 -> 15
```

- **First alias of a new day (no row yet):** the row is **INSERT**ed. Demonstrated with a **labeled runtime‑only DB action** — delete today's `daily_metric` row, then create an alias and watch the row re‑appear:

```
$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -c \
  "delete from daily_metric where date = current_date;"'   # LABELED runtime setup only
DELETE 1
$ bash /tmp/scratch_blitzy_qna/probe.sh | grep daily_metric   # BEFORE
daily_metric=0

$ curl -sS -o /dev/null -D - -b john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid token>' | grep -i '^HTTP'
HTTP/1.1 302 FOUND

$ bash /tmp/scratch_blitzy_qna/probe.sh | grep -E 'alias=|alias_audit_log=|daily_metric='  # AFTER
alias=16
alias_audit_log=16
daily_metric=1

$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc \
  "select id, date, nb_alias from daily_metric where date = current_date;"'
2|2026-07-08|1           # fresh row INSERTed (id 2), created with nb_alias=0 then incremented to 1
```

So on the **first‑of‑day** creation the touched‑table row‑count delta is `alias +1`, `alias_audit_log +1`, `daily_metric +1` (INSERT); on **every later same‑day** creation it is `alias +1`, `alias_audit_log +1`, `daily_metric +0` (UPDATE). **Either way the set of distinct tables touched is the same three.**

### 5.4 API creation re‑confirms the identical 3‑table core

```
$ bash /tmp/scratch_blitzy_qna/probe.sh   # BEFORE API call
alias=17
alias_audit_log=17
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2

$ curl -sS -o /dev/null -w 'HTTP %{http_code}\n' -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: code'
HTTP 201

$ bash /tmp/scratch_blitzy_qna/probe.sh   # AFTER API call
alias=18
alias_audit_log=18
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2
```

Same delta as the web path (`alias +1`, `alias_audit_log +1`, `daily_metric` UPDATE, all else +0), **proving both entry points share `Alias.create`** (`app/models.py:L1628-L1692`).

> **API‑transport‑only write (not part of the creation core).** API‑key authentication updates the `api_key` row's usage counters (`app/api/base.py:L30-L32`): for key `code`, `times` went `1 → 2` and `last_used` advanced. This is auth bookkeeping on `api_key`, **distinct from** the three creation tables, and does not occur on the cookie‑authenticated web path.

### 5.5 New‑row contents

```
$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc \
  "select id,email,user_id,mailbox_id,enabled,note from alias where id=14;"'
14|behave_wander222@sl.local|1|1|t|

$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc \
  "select id,user_id,alias_id,alias_email,action,message from alias_audit_log where id=14;"'
14|1|14|behave_wander222@sl.local|create|New alias created
```

- The `alias` row carries `mailbox_id=1` **directly on the row** (`app/dashboard/views/index.py:L106` sets `alias.mailbox_id = current_user.default_mailbox_id`). Because the single default mailbox is stored on the alias row itself, **no `alias_mailbox` join row is created** — that table is only used for *additional* mailboxes (`app/models.py:L2939`; `app/fake_data.py:L155-L157`).
- The `alias_audit_log` row has `action='create'` (`AliasAuditLogAction.CreateAlias = "create"`, `app/alias_audit_log_utils.py:L8`) and `message='New alias created'` (`app/models.py:L1688-L1690`).

### 5.6 What is NOT written (related‑entity absences, default flow)

Across every successful creation above, these stayed at zero‑delta, and each has a code reason:

| Table | Delta | Why (cause → effect) |
|-------|-------|----------------------|
| `sync_event` | **+0** | Event dispatch short‑circuits before the DB write because `EVENT_WEBHOOK` is unset (`app/events/event_dispatcher.py:L61-L65`); the write path `SyncEvent.create` + `NOTIFY` (`L24-L26`) is never reached. See §6. |
| `alias_mailbox` | **+0** | The single default mailbox lives on the `alias` row (`mailbox_id`); the join table is for extra mailboxes only (`app/models.py:L2939`). |
| `users` | **+0** | The partner‑flag UPDATE fires only when `FLAG_PARTNER_CREATED` is set on the alias and the user's `FLAG_CREATED_ALIAS_FROM_PARTNER` is unset (`app/models.py:L1663-L1667`); not the case in the default flow. |
| `alias_used_on` | **+0** | Written only on the **API** path when a `hostname` query param is supplied (`app/api/views/new_random_alias.py:L109-L112`); see the conditional in §7.6. |
| `job` | **+0** | Random alias creation enqueues no async job (§6). |

---

## 6. O7 — Background tasks, follow‑up events, and additional work (from the logs)

**Direct answer:** in the default configuration there is **no background or follow‑up work** after the initial write. Alias creation produces one creation log line (web path) and then event dispatch **short‑circuits**; nothing is enqueued and nothing is notified.

### 6.1 The per‑request log slice for one web creation

```
$ docker exec sl-app bash -lc "sed -n '<creation slice>' /tmp/gunicorn.log"
2026-07-08 04:31:30,909 - SL - INFO  - 810 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 04:31:30,913 - SL - DEBUG - 810 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 14 behave_wander222@sl.local> for user <User 1 John Wick john@wick.com>
2026-07-08 04:31:30,918 - SL - DEBUG - 810 - "/app/server.py:284" - after_request() -  - 172.18.0.1 POST /dashboard/ ImmutableMultiDict([]) 302, takes 0.036...
```

- **Creation log line:** `app/dashboard/views/index.py:110` — `LOG.d("create new random alias %s for user %s", …)`. This is a `LOG.d` (DEBUG). **It is emitted on the web path only**; the API success path returns `201` without an equivalent explicit creation line (verified: no `index.py:110` line appears for API calls — that handler is `new_random_alias`, which does not log a creation line on success).
- **Event short‑circuit line:** `app/events/event_dispatcher.py:62` — `LOG.i("Not sending events because webhook is not configured and allowed to be empty")`. Note it is a `LOG.i` (INFO), not DEBUG.

### 6.2 Why event dispatch short‑circuits — the three early returns

`EventDispatcher.send_event` (`app/events/event_dispatcher.py:L48-L84`) has three guard clauses; the **second** one fires here:

```
# app/events/event_dispatcher.py:L57-L70
if config.EVENT_WEBHOOK_DISABLE:                                   # L57  -> False (unset)
    LOG.i("Not sending events because webhook is disabled")
    return
if not config.EVENT_WEBHOOK and skip_if_webhook_missing:           # L61  -> True  (EVENT_WEBHOOK is None)  <-- FIRES
    LOG.i("Not sending events because webhook is not configured and allowed to be empty")   # L62 (the observed line)
    return
partner_user = EventDispatcher.__partner_user(user.id)             # L67  (not reached)
if not partner_user:
    LOG.i(f"Not sending events because there's no partner user for user {user}")
    return
```

Because `EVENT_WEBHOOK is None` (§2.2), the second `return` executes and the write path is never reached:

```
# app/events/event_dispatcher.py:L24-L26  (PostgresDispatcher.send — UNREACHED in default config)
def send(self, event: bytes):
    instance = SyncEvent.create(content=event, flush=True)                 # would INSERT sync_event
    Session.execute(f"NOTIFY {NOTIFICATION_CHANNEL}, '{instance.id}';")    # would NOTIFY simplelogin_sync_events
```

**Consequence, proven by the probe:** `sync_event` stays at `0` on every creation (§5), i.e. **no `sync_event` row and no `NOTIFY`**. The `AliasCreated` protobuf event is still *built* in `Alias.create` (`app/models.py:L1680-L1687`, fields `id, email, note, enabled=True, created_at` per `proto/event.proto`), but it is discarded at the second early return.

### 6.3 The async workers do no alias‑creation follow‑up

- **`job_runner.py`** processes a `Job` queue (`process_job` handles `JOB_ONBOARDING_*`, `JOB_BATCH_IMPORT`, mailbox deletion, etc., `app/jobs`/`job_runner.py`). Random alias creation **enqueues no `Job`** (`job` table stays `0`, §5). Running the worker briefly showed only its startup loop and **no alias‑creation work**:

```
$ timeout 8 docker exec sl-app bash -lc 'cd /app && venv/bin/python job_runner.py' 2>&1 | tail -3
# (startup / idle loop only; no alias-creation follow-up; exit via timeout)
```

- **`event_listener.py`** consumes Postgres `LISTEN/NOTIFY`. Started in dry‑run it initialized its source/sink and idled, consuming nothing (because no `NOTIFY` was ever issued):

```
$ timeout 8 docker exec sl-app bash -lc 'cd /app && venv/bin/python event_listener.py listener --dry-run' 2>&1 | tail -4
... Using PostgresEventSource ...
... Starting with ConsoleEventSink ...
... Starting to listen to events ...
# then idle; sync_event still 0
```

> The bulk job `send_alias_creation_events_for_user` exists (`app/jobs/event_jobs.py`) but is a **backfill** over *all* of a user's aliases, triggered elsewhere (not by a single alias creation) and itself calls `EventDispatcher.send_event`, which short‑circuits in this config. So even if it ran, it would emit nothing here. **[The no‑follow‑up conclusion is confirmed by observation — `job=0`, `sync_event=0`, and idle workers — not merely inferred.]**


---

## 7. O8 — Failure behavior (validation, DB, network) across response, logs, and DB

**Direct answer:** every failure branch refuses the operation **before or without committing an alias**, so on refusal the database delta is **zero**. The surface differs by transport and by cause, enumerated below. Each subsection shows the raw response, the log line, and the DB effect.

### 7.1 Free‑plan alias cap (`MAX_NB_EMAIL_FREE_PLAN = 5`)

The guard is `user.can_create_new_alias()` (`app/models.py:L867`). **Because `john@wick.com` is premium (§2.4), the cap does not apply to him** — this had to be reproduced with the runtime‑created free user `freebie@sl.local` (id 3, key `freecode`), whose free status was confirmed:

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
from app.models import User
u=User.get(3)
print(\"email=\",u.email,\"can_create_new_alias=\",u.can_create_new_alias(),
      \"lifetime_or_active_subscription=\",u.lifetime_or_active_subscription())
"'
email= freebie@sl.local can_create_new_alias= False lifetime_or_active_subscription= False
```

**API surface → HTTP 400** (guard `app/api/views/new_random_alias.py:L35-L43`):

```
$ bash /tmp/scratch_blitzy_qna/probe.sh | tr '\n' ' '     # BEFORE
alias=31 alias_audit_log=33 alias_mailbox=1 alias_used_on=1 daily_metric=1 job=0 sync_event=0 users=3

$ curl -sS -i -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: freecode'
HTTP/1.1 400 BAD REQUEST
Server: gunicorn/20.0.4
Content-Type: application/json
Content-Length: 141
...
{"error":"You have reached the limitation of a free account with the maximum of 5 aliases, please upgrade your plan to create more aliases"}

$ bash /tmp/scratch_blitzy_qna/probe.sh | tr '\n' ' '     # AFTER (identical — zero write)
alias=31 alias_audit_log=33 alias_mailbox=1 alias_used_on=1 daily_metric=1 job=0 sync_event=0 users=3
```

Log:

```
2026-07-08 04:52:45,489 - SL - DEBUG - 809 - "/app/app/api/views/new_random_alias.py:36" - new_random_alias() -  - user <User 3 Free Bee freebie@sl.local> cannot create new random alias
2026-07-08 04:52:45,490 - SL - DEBUG - 809 - "/app/server.py:284" - after_request() -  - 172.18.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 400, takes 0.013...
```

**Web surface → HTTP 302 + flash** (else branch of `can_create_new_alias`, `app/dashboard/views/index.py`):

```
$ curl -sS -i -b free.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid freebie token>'
HTTP/1.1 302 FOUND
Content-Length: 309
Location: http://localhost:7777/dashboard/?query=&sort=&filter=&page=0

$ curl -sS -b free.cookies http://localhost:7777/dashboard/ | grep -o 'You need to upgrade your plan to create new alias.'
You need to upgrade your plan to create new alias.
```

- The web redirect goes to the fall‑through `/dashboard/?query=&sort=&filter=&page=0` — **note the absence of `highlight_alias_id`**, which distinguishes a refusal from a success (§4.1). The flash *"You need to upgrade your plan to create new alias."* renders on the followed page. Alias count was unchanged (`31 → 31`).

### 7.2 Trashed‑alias reuse → `AliasInTrashError` (two distinct surfaces)

`Alias.create` refuses to re‑mint an email that sits in the trash tables, raising `AliasInTrashError` (`app/models.py:L1647-L1652`) — reproduced verbatim with line numbers:

```python
# app/models.py:L1647-L1652 (verbatim)
1647:         # make sure alias is not in global trash, i.e. DeletedAlias table
1648:         if DeletedAlias.get_by(email=email):
1649:             raise AliasInTrashError
1650:
1651:         if DomainDeletedAlias.get_by(email=email):
1652:             raise AliasInTrashError
```

**Surface A — random endpoint's hostname branch (raises, then catches, then falls back → still 201).** First delete a website‑derived alias so its email lands in the (domain) trash, then re‑request it:

```
$ curl -sS -i -X DELETE http://localhost:7777/api/aliases/21 -H 'Authentication: code' | head -1
HTTP/1.1 200 OK
# body: {"deleted":true}
$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc \
  "select id,email from domain_deleted_alias where email like '\''groupon%'\'';"'
1|groupon@old.com                 # groupon@old.com is now trashed

$ curl -sS -i -X POST 'http://localhost:7777/api/alias/random/new?hostname=groupon.com' -H 'Authentication: code'
HTTP/1.1 201 CREATED
...
{"alias":"ibidem_chilli702@sl.local","creation_date":"2026-07-08 04:44:52+00:00", ... ,"id":27, ...}
```

The response is **`201` with a *random* alias (`ibidem_chilli702@sl.local`, id 27), not `groupon@old.com`** — the trashed email was refused and the endpoint fell back to a random one. The log shows the exact exception path:

```
2026-07-08 04:44:52,505 - SL - INFO  - 810 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 1 John Wick john@wick.com> has deleted alias <Alias 21 groupon@old.com>
2026-07-08 04:44:52,551 - SL - INFO  - 810 - "/app/app/alias_utils.py:360" - delete_alias() -  - Moving <Alias 21 groupon@old.com> to domain 2 trash <DomainDeletedAlias 1 groupon@old.com>
2026-07-08 04:44:52,720 - SL - DEBUG - 809 - "/app/app/api/views/new_random_alias.py:55" - new_random_alias() -  - Use groupon.com to create new alias
2026-07-08 04:44:52,746 - SL - DEBUG - 809 - "/app/app/api/views/new_random_alias.py:82" - new_random_alias() -  - create new alias groupon@old.com
2026-07-08 04:44:52,749 - SL - INFO  - 809 - "/app/app/api/views/new_random_alias.py:92" - new_random_alias() -  - Alias groupon@old.com is in trash
2026-07-08 04:44:52,788 - SL - DEBUG - 809 - "/app/server.py:284" - after_request() -  - 172.18.0.1 POST /api/alias/random/new ImmutableMultiDict([('hostname', 'groupon.com')]) 201, takes 0.076...
```

The line `new_random_alias.py:92 Alias groupon@old.com is in trash` is the `except AliasInTrashError:` handler at `app/api/views/new_random_alias.py:L91-L93`, catching the raise from `app/models.py:L1651-L1652` (the `DomainDeletedAlias` branch — `groupon@old.com` is a custom‑domain alias, so the delete moved it to *domain* trash, confirmed by the log line `Moving <Alias 21 groupon@old.com> to domain 2 trash`). DB effect: the delete + random‑create left `alias` net **0** (−1 for the deleted alias 21, +1 for the random alias 27) and `alias_audit_log` **+2** (one `delete`, one `create`), confirmed:

```
$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc \
  "select id,alias_id,action,message from alias_audit_log order by id desc limit 2;"'
28|27|create|New alias created
27|21|delete|Alias deleted by user action
```

**Surface B — custom‑alias endpoint pre‑checks trash → HTTP 409 (client‑visible error).** The v2/v3 custom endpoints explicitly check the trash tables *before* calling `Alias.create` and return `409` (`app/api/views/new_custom_alias.py:L83-L88`). Using a real signed suffix obtained from the options endpoint:

```
$ curl -sS 'http://localhost:7777/api/v5/alias/options' -H 'Authentication: code' | python3 -m json.tool | grep -A1 '"suffix": "@old.com"'
            "signed_suffix": "@old.com.ak3WGQ.k0k4Hxln6JvLuHiTRabf5G8d5xM",
            "suffix": "@old.com"

$ bash /tmp/scratch_blitzy_qna/probe.sh | tr '\n' ' '     # BEFORE
alias=26 alias_audit_log=28 alias_mailbox=1 alias_used_on=1 daily_metric=1 job=0 sync_event=0 users=3

$ curl -sS -i -X POST http://localhost:7777/api/v2/alias/custom/new -H 'Authentication: code' \
       -H 'Content-Type: application/json' \
       -d '{"alias_prefix":"groupon","signed_suffix":"@old.com.ak3WGQ.k0k4Hxln6JvLuHiTRabf5G8d5xM"}'
HTTP/1.1 409 CONFLICT
Content-Type: application/json
Content-Length: 49
...
{"error":"alias groupon@old.com already exists"}

$ bash /tmp/scratch_blitzy_qna/probe.sh | tr '\n' ' '     # AFTER (identical — zero write)
alias=26 alias_audit_log=28 alias_mailbox=1 alias_used_on=1 daily_metric=1 job=0 sync_event=0 users=3
```

Log: `new_custom_alias.py:87 full alias already used groupon@old.com` then `... POST /api/v2/alias/custom/new ... 409`. **Zero DB writes.** So the trash guard surfaces as a caught‑and‑fallback (`201`, random) on the random endpoint and as an explicit **`409`** on the custom endpoint.

### 7.3 Invalid CSRF (web) → flash "Invalid request" + 302

CSRF validation failure is handled at `app/dashboard/views/index.py:L88-L90` (`if not csrf_form.validate(): flash("Invalid request","warning"); return redirect(request.url)`):

```
$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc "select count(*) from alias;"'   # BEFORE
31
$ curl -sS -i -b jcsrf.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=GARBAGE_INVALID_TOKEN_xxx'
HTTP/1.1 302 FOUND
Content-Length: 271
Location: http://localhost:7777/dashboard/

$ curl -sS -b jcsrf.cookies http://localhost:7777/dashboard/ | grep -o 'Invalid request'
Invalid request
$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc "select count(*) from alias;"'   # AFTER
31
```

- **`302 FOUND`** with `Location: http://localhost:7777/dashboard/` — exactly `request.url` (the dashboard root, **no** query string and **no** `highlight_alias_id`), which distinguishes it from both the success redirect (§4.1) and the free‑plan refusal redirect (§7.1). The flash *"Invalid request"* renders on the followed page; alias count unchanged (`31 → 31`, **zero write**).

### 7.4 Invalid API `mode` → HTTP 400

`mode` is validated at `app/api/views/new_random_alias.py:L98-L104` (`word`/`uuid`, else 400):

```
$ curl -sS -i -X POST 'http://localhost:7777/api/alias/random/new?mode=banana' -H 'Authentication: code'
HTTP/1.1 400 BAD REQUEST
Content-Type: application/json
Content-Length: 47
...
{"error":"banana must be either word or uuid"}
```

Log: `... POST /api/alias/random/new ImmutableMultiDict([('mode', 'banana')]) 400`. DB delta **zero** (probe identical before/after). For contrast, the valid modes both succeed: `?mode=word → 201` (`scruff_tinker385@sl.local`, id 19) and `?mode=uuid → 201` (uuid‑style, id 20).

### 7.5 Rate limiting → HTTP 429

The endpoints carry `@limiter.limit(ALIAS_LIMIT)` where `ALIAS_LIMIT="100/day;50/hour;5/minute"` (`app/config.py:L448`). Driving the API past the per‑minute bucket:

```
$ for i in $(seq 1 20); do
    curl -sS -o /dev/null -w '%{http_code} ' -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: code'
  done; echo
201 201 201 201 201 429 429 429 429 429 429 429 429 429 429 429 429 429 429 429
```

The first **5** requests returned `201`, the remaining **15** returned `429` — a clean enforcement of the `5/minute` bucket. Raw 429:

```
$ curl -sS -i -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: code'
HTTP/1.1 429 TOO MANY REQUESTS
Server: gunicorn/20.0.4
Content-Type: application/json
Content-Length: 32
...
{"error":"Rate limit exceeded"}
```

Log (the custom handler at `server.py:364`):

```
2026-07-08 04:47:15,236 - SL - WARNING - 809 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/alias/random/new, user:<flask_login.mixins.AnonymousUserMixin object ...>
2026-07-08 04:47:15,237 - SL - DEBUG   - 809 - "/app/server.py:284" - after_request() -  - 172.18.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 429, takes 0.0008...
```

- **DB effect of the burst:** only the 5 successful calls created aliases (`alias 26 → 31`, `alias_audit_log 28 → 33`); the `429`s wrote **nothing**.
- **Key function** (`app/extensions.py:L14-L23`): the limiter keys by `userid:{id}` when `current_user.is_authenticated`, else `ip:{addr}`. On the API path there is no login session, so `current_user` is anonymous (the log's `user:<AnonymousUserMixin>` confirms this) and the bucket is keyed by **client IP**.
- **There are three rate‑limit layers; only one is active in this config:**

| Layer | Mechanism | Runtime state |
|-------|-----------|---------------|
| L1 (active) | flask‑limiter `@limiter.limit(ALIAS_LIMIT)` (`config.py:L448`) | **Active**, in‑memory storage (no `MEM_STORE_URI`). This produced the observed `429`. |
| L2 (no‑op) | in‑`Alias.create` Redis token bucket `rate_limiter.check_bucket_limit` (`app/models.py:L1632-L1641`, `ALIAS_CREATE_RATE_LIMIT_{FREE,PAID}`) | **No‑op** — `rate_limiter.lock_redis is None`, so it returns immediately (`app/rate_limiter.py:L28-L29`). |
| L3 (no‑op) | `@parallel_limiter.lock(name="alias_creation")` (`app/parallel_limiter.py`) | **No‑op** — `parallel_limiter.lock_redis is None`, so the decorator is a pass‑through (`L51-L52`). |

Runtime proof of the two no‑ops:

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
import app.rate_limiter as rl, app.parallel_limiter as pl
from app import config
print(\"rate_limiter.lock_redis =\", rl.lock_redis)
print(\"parallel_limiter.lock_redis =\", pl.lock_redis)
print(\"MEM_STORE_URI =\", config.MEM_STORE_URI)
"'
rate_limiter.lock_redis = None
parallel_limiter.lock_redis = None
MEM_STORE_URI = None
```

Both are `None` because `MEM_STORE_URI` is unset, so `server.py`'s `initialize_redis_services` never wired them. **Nuance:** flask‑limiter's in‑memory storage is *per‑gunicorn‑worker* (2 workers here), so under different routing the effective per‑minute allowance could be as high as ~2× nominal; in this run the observed enforcement was a clean 5 before the first `429`. `DISABLE_RATE_LIMIT=False` (§2.2), so the limiter is enabled. Since `john` is premium, layer L2 would use the **PAID** bucket if it were active (`app/models.py:L1634-L1641`) — but it is a no‑op here regardless.

> **"Network" failure mode.** The prompt's "network" failure class maps, in this app, to (a) rate limiting / throttling — the `429` above — and (b) the event/`NOTIFY` egress, which in default config never fires (§6). There is no outbound email either (`NOT_SEND_EMAIL=true`). No genuine socket‑level network dependency is on the alias‑creation happy path, so there is no additional network‑error surface to trigger beyond throttling. **[This scoping is inferred from configuration + code; the throttling surface itself is observed.]**

### 7.6 Conditional writes (caveats) — when the "extra" tables would be touched

These are **not** part of the default 3‑table flow; each is gated and was either exercised or shown absent:

- **`alias_used_on` (API only, `hostname` supplied)** — **exercised.** `POST /api/alias/random/new?hostname=groupon.com` wrote one `alias_used_on` row (`app/api/views/new_random_alias.py:L109-L112`):

```
$ curl -sS -o /dev/null -w 'HTTP %{http_code}\n' -X POST 'http://localhost:7777/api/alias/random/new?hostname=groupon.com' -H 'Authentication: code'
HTTP 201
$ docker exec sl-app bash -lc 'PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tAc \
  "select id,alias_id,hostname,user_id from alias_used_on order by id desc limit 1;"'
2|27|groupon.com|1
```

  When `user.include_website_in_one_click_alias` is true (it is for john), this branch *also* attempts a website‑derived alias (e.g. `groupon@old.com`) before writing `alias_used_on` (`new_random_alias.py:L54-L93`). This is an **API‑only** write and never occurs on the web random‑alias path.

- **`users` UPDATE (partner‑created flag)** — **not hit; observed absent.** Fires only if `new_alias.flags & FLAG_PARTNER_CREATED > 0 and user.flags & FLAG_CREATED_ALIAS_FROM_PARTNER == 0` (`app/models.py:L1663-L1667`). The seeded/free users are not partner‑created, so `users` stayed at zero‑delta in every run. **[Read‑verified + observed‑absent.]**

- **`sync_event` row + `NOTIFY`** — **not hit; observed absent.** Occurs only when `EVENT_WEBHOOK` is set *and* the user is a partner user (`app/events/event_dispatcher.py:L61-L70`, write at `L24-L26`). With `EVENT_WEBHOOK` unset the second early return fires (§6), so `sync_event` stayed `0` on every creation. **[Read‑verified + observed‑absent.]**


---

## 8. Nuance, caveats, and configuration boundaries

### 8.1 The single most important gotcha — the happy‑path user is premium

`john@wick.com` is **premium** (active monthly subscription, `app/fake_data.py:L106-L119`), so `can_create_new_alias()` short‑circuits `True` (`app/models.py:L867`, `L746`) and the free‑plan cap **cannot** be observed with him. All cap testing used the runtime‑created free user `freebie@sl.local` (§7.1). Any reader reproducing this must not expect the seeded user to hit the cap.

### 8.2 Default‑configuration invariants (the boundary within which the "3 tables / no background work" answer holds)

- `EVENT_WEBHOOK` unset → event dispatch short‑circuits → **no `sync_event`, no `NOTIFY`, no background follow‑up** (§6).
- `MEM_STORE_URI` unset → Redis limiters not initialized → the in‑`Alias.create` bucket and the `alias_creation` parallel lock are **no‑ops**; only flask‑limiter (in‑memory) is active (§7.5).
- Seeded user has a single default mailbox → **no `alias_mailbox`** row (§5.5).
- `NOT_SEND_EMAIL=true` → no outbound email is transmitted on any path.

### 8.3 When the conditional writes *would* occur (outside the default flow)

- **4th table `alias_used_on`** — API path only, when `?hostname=` is supplied (§7.6). Exercised: `201` + one `alias_used_on` row.
- **`users` UPDATE** — only for partner‑created aliases (`FLAG_PARTNER_CREATED`), `app/models.py:L1663-L1667`. Not reachable in the default flow.
- **`sync_event` + `NOTIFY`** — only when `EVENT_WEBHOOK` is set *and* the user is a partner user, `app/events/event_dispatcher.py:L61-L70` / `L24-L26`. Not reachable in the default flow.
- **API‑transport `api_key` update** — the API auth decorator bumps `api_key.times`/`last_used` (`app/api/base.py:L30-L32`); this is auth bookkeeping, not part of the alias‑creation core, and is absent on the web path (§5.4).

### 8.4 Repository‑unchanged & cleanup verification

All observation scripts lived under `/tmp/scratch_blitzy_qna` (host), never in the repository, and were deleted at completion; the app's own `/tmp/gunicorn.log` inside the container was read but left intact. The repository was confirmed unchanged apart from this new document (captured live at cleanup):

```
$ git status --porcelain
?? blitzy/

$ git status --porcelain -uall     # expand the untracked dir to prove a single file
?? blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --stat                  # tracked files — no output = zero changes
$ git diff --name-only             # tracked files — no output = zero changes

$ git submodule status
# (no .gitmodules; no submodules)
```

> The running database was mutated during observation (new aliases, a runtime free user, a trashed alias, an incremented API‑key counter, a temporarily deleted `daily_metric` row). That is ephemeral runtime state; **no tracked repository file was modified, added, or deleted** other than the answer document.

---

## 9. Coverage pass — every named item answered

Re‑reading the original question and confirming each named item is addressed:

| Named item asked | Answered in | Direct result |
|------------------|-------------|---------------|
| **Frontend request** | §3 | `application/x-www-form-urlencoded` `POST /dashboard/` with `form-name=create-random-email` + `csrf_token` (+ optional `generator_scheme`) |
| — its status codes / payloads / metadata (of response) | §4 | web `302` (+ `Location`, flash cookie); API `201` (+ JSON, headers) |
| **Backend response** | §4 | web `302` redirect; API `201` + 17‑key JSON, every key mapped to a serializer line |
| **Database changes** | §5 | 3 tables: `alias` INSERT, `daily_metric` INSERT/UPDATE, `alias_audit_log` INSERT |
| — which records inserted/updated | §5.5, §5.3 | new `alias` + `alias_audit_log` rows shown; `daily_metric` INSERT‑vs‑UPDATE shown |
| — how many tables touched | §5.1–§5.4 | **exactly 3**, stable across ≥2 runs |
| — are related entities created | §5.6, §7.6 | not in default flow (`alias_mailbox`/`users`/`sync_event`/`alias_used_on` all +0); `alias_used_on` only for API `hostname` |
| **Background tasks / follow‑up events / additional work** | §6 | none in default config; creation `LOG.d` + event short‑circuit `LOG.i`; no `NOTIFY`, no job |
| — inspect logs | §6.1, throughout | raw log slices shown for creation and every error branch |
| **Failure — validation** | §7.3, §7.4 | invalid CSRF → `302` + "Invalid request"; invalid `mode` → `400` |
| **Failure — DB** | §7.2 | trashed‑alias reuse → `AliasInTrashError` (caught → `201` random fallback / custom → `409`) |
| **Failure — network (throttling/egress)** | §7.5, §6 | rate limit → `429`; event egress short‑circuited (no `NOTIFY`) |
| — free‑plan limit (an implied failure) | §7.1 | API `400`, web `302` + upgrade flash; zero DB write |
| — how each error surfaces in response / logs / DB | §7 (all) | each branch shows raw response + log line + zero‑delta DB |

All "e.g. / such as / including" items (status codes, payloads, metadata; records inserted/updated; table count; related entities; validation + DB + network failure modes) are covered above.

---

## 10. Evidence index (`file:line` references used)

- **Entry points & app factory:** `server.py:L139` (`create_app`), `server.py:L251-L255` (root `/` redirect), `server.py:L284` (`after_request` request log), `server.py:L364` (`rate_limited` 429 handler), `server.py:L588` (dev bind 7777); `wsgi.py` (gunicorn entry).
- **Auth (web/login):** `app/auth/views/login.py:L21-L25`.
- **Web alias handler:** `app/dashboard/views/index.py:L55-L121` — CSRF fail `L88-L90`; `Alias.create_new_random` `L104`; `mailbox_id` `L106`; `Session.commit` `L108`; creation `LOG.d` `L110`; success flash `L111`; `302` redirect `L113-L121`; upgrade‑flash else branch.
- **Frontend template:** `templates/dashboard/index.html:L50-L58` (main button), `L68-L76` (word), `L78-L86` (uuid).
- **JSON API (random):** `app/api/views/new_random_alias.py` — route `L21`, `@limiter.limit` `L22`, `@require_api_auth` `L23`, `@parallel_limiter.lock` `L24`; free‑plan `400` guard `L35-L43` (log `L36`); hostname branch `L53-L93` (`Alias.create` `L84-L90`, `except AliasInTrashError` `L91-L93`); `mode` `400` `L98-L104`; `Session.commit` `L107`; `alias_used_on` `L109-L112`; `201` `L114-L116` (top‑level `alias` `L115`).
- **JSON API (custom):** `app/api/views/new_custom_alias.py:L83-L88` (trash pre‑check → `409`, log `L87`).
- **API auth:** `app/api/base.py:L16-L34` (`Authentication` header, `ApiKey.get_by`, usage update `L30-L32`, `g.user` `L34`).
- **Serializer:** `app/api/serializer.py:L55-L93` (`serialize_alias_info_v2`; dict literal `L56-L79`, keys `id`=`L58` … `pinned`=`L78`; `latest_activity` populate block `L80-L92`).
- **Models:** `app/models.py` — `Alias.create` `L1628-L1692` (`AliasInTrashError` `L1647-L1652`, `alias` INSERT `L1660`, `daily_metric` `L1661`, partner‑flag `users` `L1663-L1667`, event build `L1680-L1687`, `send_event` `L1687`, `emit_alias_audit_log` `L1688-L1690`); `create_new_random` `L1721`; `can_create_new_alias` `L867`; `lifetime_or_active_subscription` `L746`; `DailyMetric.get_or_create_today_metric` `L3280`; `AliasAuditLog` `L3810`; `AliasMailbox` `L2939`; rate bucket select `L1632-L1641`.
- **Audit utils:** `app/alias_audit_log_utils.py:L8` (`CreateAlias="create"`), `L18-L32` (`emit_alias_audit_log`).
- **Event pipeline:** `app/events/event_dispatcher.py:L24-L26` (`SyncEvent.create` + `NOTIFY`), `L48-L84` (`send_event` three early returns; the one that fires `L61-L65`, log at `L62`); `proto/event.proto` / `app/events/generated/event_pb2.py` (`AliasCreated` fields).
- **Alias delete (trash setup):** `app/api/views/alias.py:L152-L173` (`DELETE`), `app/alias_utils.py:L336-L372` (`delete_alias` → `DeletedAlias`/`DomainDeletedAlias`).
- **Rate limiting / concurrency:** `app/config.py:L448` (`ALIAS_LIMIT`), `L554-L558` (`ALIAS_CREATE_RATE_LIMIT_*`); `app/extensions.py:L14-L23` (limiter key func); `app/rate_limiter.py:L28-L29` (no‑op guard); `app/parallel_limiter.py:L51-L52` (no‑op guard).
- **Config / seeding:** `app/config.py:L124` (`MAX_NB_EMAIL_FREE_PLAN`), `L568` (`MEM_STORE_URI`), `L602` (`DISABLE_RATE_LIMIT`), `L612` (`EVENT_WEBHOOK`), `L616` (`EVENT_WEBHOOK_DISABLE`); `app/fake_data.py:L44-L55` (john), `L106-L119` (subscriptions), `L121-L125` (API keys `code`/`codeFF`), `L155-L157` (`AliasMailbox` extra‑mailbox only).
- **Background workers:** `event_listener.py` (LISTEN/NOTIFY consumer), `job_runner.py` (job queue), `app/jobs/event_jobs.py` (`send_alias_creation_events_for_user` backfill).
- **Logging:** `app/log.py` (`SL` logger, `LOG.d/i/w/e`).

---

*End of investigation. This document is the sole persistent artifact; all temporary observation scripts were removed and the repository left byte‑for‑byte unchanged apart from this file.*

