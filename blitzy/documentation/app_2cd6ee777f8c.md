# SimpleLogin Custom-Alias Creation — Signed-Suffix Verification & Alias-Creation Limits

> **Diagnostic Q&A — branch `app_2cd6ee777f8c`**
>
> This document explains, with code‑level evidence, exactly how SimpleLogin verifies
> **signed alias suffixes** and how it enforces **alias‑creation limits** on the custom‑alias
> creation flow. It exists to help diagnose *"intermittent validation failures that don't
> match the expected behavior."*

## 0. Scope, Method, and How to Read This Document

**Scope.** This investigation is limited to the **custom‑alias creation flow** — the two API
endpoints `POST /api/v2/alias/custom/new` and `POST /api/v3/alias/custom/new`, the shared
signed‑suffix verifier in `app/alias_suffix.py`, and the three independent limiting mechanisms
that gate alias creation. Unrelated SimpleLogin features (random‑alias creation, the email
handler, OAuth/OIDC, PGP, custom‑domain verification, billing) are intentionally out of scope.

**Method (read‑only, code‑as‑truth).** Every factual claim below is grounded in the source
code with `file:line` citations, and the key behaviors were additionally **reproduced live**
against the running application (Flask test client + a throwaway script, both discarded). No
source file was modified, added, or deleted — the SimpleLogin codebase remains byte‑for‑byte
unchanged. Discovered anomalies are **reported as findings, not fixed**.

**The six questions answered.**

| # | Question | Section |
|---|----------|---------|
| **Q1** | What HTTP status codes / error bodies are returned for an invalid or expired signed suffix? | [§2](#2-q1q6--failure-responses--rejection-conditions) |
| **Q2** | What validation log entries are written during signed‑suffix verification? | [§3](#3-q2--server-console-validation-log-entries) |
| **Q3** | Do rate‑limit headers appear in responses, and what are their values? | [§4](#4-q3--rate-limit-headers--the-three-limiting-mechanisms) |
| **Q4** | On a successful creation, what quota‑related checks execute? | [§5](#5-q4q5--success-path-quota-checks--what-is-logged) |
| **Q5** | What specific values are logged when verifying creation permission? | [§5](#5-q4q5--success-path-quota-checks--what-is-logged) |
| **Q6** | Which component validates suffixes / enforces limits, and what triggers rejection? | [§1](#1-execution-path-overview--the-validating-component) & [§2](#2-q1q6--failure-responses--rejection-conditions) |

**Pinned dependency versions** (from `poetry.lock`, confirmed at runtime). These versions are
load‑bearing for the analysis — in particular the `itsdangerous` exception hierarchy:

| Package | Version | Relevance |
|---------|---------|-----------|
| flask | 1.1.2 | `jsonify` JSON responses; the `api` blueprint |
| flask-limiter | 1.4 | HTTP route rate limiting; **headers off by default** |
| itsdangerous | 1.1.0 | `TimestampSigner`; `SignatureExpired ⊂ BadTimeSignature ⊂ BadSignature` |
| werkzeug | 1.0.1 | `exceptions.TooManyRequests` (HTTP 429) |
| redis | 4.6.0 | Backs the session store, concurrency lock, and bucket quota |
| limits | 1.5.1 | `RedisStorage` abstraction used by the limiters/lock |
| sqlalchemy | 1.3.24 | ORM persisting `Alias` rows |
| newrelic | 8.8.0 | Records the `BucketRateLimit` custom event on a bucket breach |

---

> ## ⭐ TL;DR — The Headline Finding
>
> **A *tampered* or *malformed* signed suffix produces the same response as an *expired* one:
> `HTTP 412 {"error":"Alias creation time is expired, please retry"}`.**
>
> The verifier `check_suffix_signature()` catches `itsdangerous.BadSignature`
> [`app/alias_suffix.py:L41`], and in `itsdangerous 1.1.0` the *expired* exception
> (`SignatureExpired`) and the *tampered* exception (`BadTimeSignature`) are **both subclasses
> of `BadSignature`**. So both are swallowed identically and mapped to **412 "expired"**. The
> endpoint's `except Exception → 400 "Tampered suffix"` branch is therefore **effectively
> unreachable** for ordinary tampering. This 412‑instead‑of‑expected‑400 behavior is the single
> most likely explanation for "validation failures that don't match the expected behavior." See
> [§2.2](#22-key-finding--expired-and-tampered-suffixes-both-return-412).

---

## 1. Execution-Path Overview & the Validating Component

This section answers the **"which component"** half of **Q6** and frames everything that follows.

### 1.1 Where the 600-second clock starts — suffix issuance

Before a client can create a custom alias, it must obtain a **signed suffix**. Suffixes are
issued by the alias‑options endpoints, which call `get_alias_suffixes()`:

- `options_v4()` builds `ret["suffixes"]` as `[suffix, signed_suffix]` pairs
  [`app/api/views/alias_options.py:L69,L72`].
- `options_v5()` returns objects `{suffix, signed_suffix, is_custom, is_premium}`
  [`app/api/views/alias_options.py:L140,L143-150`].

Both delegate to `get_alias_suffixes()` [`app/alias_suffix.py:L94-192`], which signs every
suffix with `signer.sign(suffix).decode()` [`app/alias_suffix.py:L114,L128,L155,L185`]. The
signer is module‑level and time‑based:

```python
# app/alias_suffix.py:L11
signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
```

`CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` [`app/config.py:L201`]. Because
`TimestampSigner` embeds the issuance timestamp into the signature, **the 600‑second validity
window begins the moment the options endpoint signs the suffix** — not when the user submits
the form.

### 1.2 The two creation endpoints

| Endpoint | Function | Lines |
|----------|----------|-------|
| `POST /api/v2/alias/custom/new` | `new_custom_alias_v2` | `app/api/views/new_custom_alias.py:L28-112` |
| `POST /api/v3/alias/custom/new` | `new_custom_alias_v3` | `app/api/views/new_custom_alias.py:L115-235` |

The `/api` prefix comes from
`api_bp = Blueprint(name="api", import_name=__name__, url_prefix="/api")`
[`app/api/base.py:L11`].

**Decorator order is identical on both endpoints** (top → bottom)
[v2 `L28-31`, v3 `L115-118`]:

```python
@api_bp.route("/v2/alias/custom/new", methods=["POST"])   # routing
@limiter.limit(ALIAS_LIMIT)                                # (1) Flask-Limiter HTTP limit
@require_api_auth                                          # API key / session auth
@parallel_limiter.lock(name="alias_creation")             # (3) concurrency lock (per current-user-or-IP)
def new_custom_alias_v2():
    ...
```

Authentication is performed by `require_api_auth` [`app/api/base.py:L52-60`], which reads the
API key from the **`Authentication` HTTP header** [`app/api/base.py:L17`].

### 1.3 The validating component (answer to Q6 "which component")

> **Signed‑suffix verification is performed by `app/alias_suffix.py`** — specifically
> `check_suffix_signature()` [`L37-42`] and `verify_prefix_suffix()` [`L45-91`] — invoked from
> the two endpoints in `app/api/views/new_custom_alias.py`.
>
> **Alias‑creation limits are enforced by THREE separate mechanisms** (detailed in
> [§4.2](#42-three-independent-limiting-mechanisms--do-not-conflate)):
> 1. the Flask‑Limiter HTTP route limit (`@limiter.limit(ALIAS_LIMIT)`),
> 2. the per‑user/per‑window **bucket quota** (`check_bucket_limit()` inside `Alias.create()`),
> 3. the **concurrency lock** (`@parallel_limiter.lock(name="alias_creation")`), keyed per current‑user‑or‑IP.

On success, `Alias.create()` [`app/models.py:L1627+`] runs the bucket quota *before* persisting
the row.

### 1.4 End-to-end request flow

```mermaid
flowchart TD
    Issue["GET /api/v4|v5/alias/options<br/>get_alias_suffixes() → signer.sign()<br/>(600s clock STARTS here)"] --> Post["POST /api/v2|v3/alias/custom/new"]
    Post --> L1["@limiter.limit(ALIAS_LIMIT)<br/>Flask-Limiter HTTP limit"]
    L1 -->|breach| R429h["429 'Rate limit exceeded'"]
    L1 --> L2["@require_api_auth<br/>API key / session"]
    L2 --> L3["@parallel_limiter.lock('alias_creation')<br/>Redis SET NX, 5s TTL<br/>(keyed per current-user or IP)"]
    L3 -->|concurrent in-flight| R429c["429 'Rate limit exceeded'"]
    L3 --> Quota["user.can_create_new_alias()?"]
    Quota -->|False| R400a["400 'You have reached the limitation...'"]
    Quota -->|True| Body["body present & well-formed?<br/>(v3: prefix/mailbox checks)"]
    Body -->|no| R400b["400 (various: empty body, bad format,<br/>prefix, mailbox_ids, mailbox)"]
    Body -->|yes| Sig["check_suffix_signature(signed_suffix)<br/>TimestampSigner.unsign(max_age=600)"]
    Sig -->|None: expired OR tampered| R412["412 'Alias creation time is expired, please retry'"]
    Sig -->|non-BadSignature exc (rare)| R400c["400 'Tampered suffix'"]
    Sig -->|valid| Verify["verify_prefix_suffix()"]
    Verify -->|False| R400d["400 'wrong alias prefix or suffix'"]
    Verify -->|True| Exists["alias / deleted-alias collision?"]
    Exists -->|Yes| R409["409 'alias ... already exists'"]
    Exists -->|No| Dots["'..' in full alias?"]
    Dots -->|Yes| R400e["400 '2 consecutive dot signs...'"]
    Dots -->|No| Create["Alias.create() → check_bucket_limit() × N"]
    Create -->|bucket exceeded| R429b["429 'Rate limit exceeded'"]
    Create -->|ok| R201["201 alias payload"]
```

---

## 2. Q1/Q6 — Failure Responses & Rejection Conditions

### 2.1 The signed-suffix verifier

```python
# app/alias_suffix.py:L37-42
def check_suffix_signature(signed_suffix: str) -> Optional[str]:
    # hypothesis: user will click on the button in the 600 secs
    try:
        return signer.unsign(signed_suffix, max_age=600).decode()   # L40
    except itsdangerous.BadSignature:                                # L41
        return None                                                  # L42
```

- `max_age=600` → a signed suffix is valid for **600 seconds (10 minutes)** after issuance
  [`app/alias_suffix.py:L40`].
- The function catches `itsdangerous.BadSignature` and returns `None`
  [`app/alias_suffix.py:L41-42`].

After a non‑`None` suffix is returned, `verify_prefix_suffix()` [`app/alias_suffix.py:L45-91`]
performs a second, semantic check: that the suffix's domain is one of the user's available
alias domains and that SimpleLogin‑domain suffixes start with `.`.

### 2.2 KEY FINDING — Expired and Tampered Suffixes Both Return 412

In **itsdangerous 1.1.0** the exception hierarchy is:

```text
SignatureExpired  ⊂  BadTimeSignature  ⊂  BadSignature  ⊂  BadData  ⊂  Exception
```

This was confirmed directly at runtime — `SignatureExpired.__mro__` is
`[SignatureExpired, BadTimeSignature, BadSignature, BadData, Exception, ...]`, and both
`issubclass(SignatureExpired, BadSignature)` and `issubclass(BadTimeSignature, BadSignature)`
are `True`.

Therefore the single `except itsdangerous.BadSignature` in `check_suffix_signature()` swallows
**both** failure modes:

- an **expired** suffix → raises `SignatureExpired` (older than 600 s), and
- a **tampered / malformed** suffix → raises `BadTimeSignature` / `BadSignature`,

returning `None` in **both** cases. The endpoint maps `None` to:

> **HTTP 412**, body `{"error":"Alias creation time is expired, please retry"}`
> [`app/api/views/new_custom_alias.py:L71-73` (v2), `L186-188` (v3)].

**Consequence — the `400 "Tampered suffix"` branch is effectively unreachable.** The
`except Exception:` guard around `check_suffix_signature()` [v2 `L74-76`, v3 `L189-191`] can
only fire if `check_suffix_signature()` raises an exception that is *not* a `BadSignature`
subclass. But ordinary tampering raises exactly a `BadSignature` subclass, which is already
caught *inside* `check_suffix_signature()`. No exception propagates to the endpoint's
`try/except`, so the `400 "Tampered suffix"` response is never produced for normal
tampered/malformed input.

**Live reproduction (read‑only, throwaway).** Submitting a deliberately corrupted suffix and a
garbage string to `POST /api/v2/alias/custom/new` both returned `412` with the *expired*
message, identical to an actually‑expired suffix:

```text
VALID    status= 201
TAMPERED status= 412 body= {'error': 'Alias creation time is expired, please retry'}
GARBAGE  status= 412 body= {'error': 'Alias creation time is expired, please retry'}
```

> **Why this matters for the diagnosis.** A caller who tampers with — or simply sends a stale,
> truncated, or wrongly‑encoded — `signed_suffix` receives a *412 "expired"* message rather than
> a *400 "bad request / tampered"*. Anyone reading the response (or the logs — see
> [§3](#3-q2--server-console-validation-log-entries)) will be told the suffix *expired*, even
> when the real problem is a malformed value. **This is documented as a finding only; per the
> task's read‑only mandate, no source change is proposed.**

### 2.3 Full rejection table — `POST /api/v2/alias/custom/new`

Conditions are listed in **execution order**. All error bodies use the JSON shape
`{"error": "<message>"}` returned as a `(jsonify(...), <status>)` tuple. Evidence:
`app/api/views/new_custom_alias.py`.

| # | Condition (code) | Status | Literal `error` body | Log call |
|---|------------------|--------|----------------------|----------|
| 1 | `not user.can_create_new_alias()` (`L48`) | **400** | `You have reached the limitation of a free account with the maximum of {MAX_NB_EMAIL_FREE_PLAN} aliases, please upgrade your plan to create more aliases` (`L51-53`) | `LOG.d("user %s cannot create any custom alias", user)` (`L49`) |
| 2 | empty JSON body — `if not data` (`L61`) | **400** | `request body cannot be empty` (`L62`) | — |
| 3 | `check_suffix_signature()` → `None` (`L70-71`) | **412** | `Alias creation time is expired, please retry` (`L73`) | `LOG.w("Alias creation time expired for %s", user)` (`L72`) |
| 4 | `except Exception` (rare / unreachable, see §2.2) (`L74`) | **400** | `Tampered suffix` (`L76`) | `LOG.w("Alias suffix is tampered, user %s", user)` (`L75`) |
| 5 | `not verify_prefix_suffix(...)` (`L78`) | **400** | `wrong alias prefix or suffix` (`L79`) | `LOG.e(...)` inside `verify_prefix_suffix` (see [§3](#3-q2--server-console-validation-log-entries)) |
| 6 | alias / deleted‑alias / domain‑deleted‑alias collision (`L82-86`) | **409** | `alias {full_alias} already exists` (`L88`) | `LOG.d("full alias already used %s", full_alias)` (`L87`) |
| 7 | `".." in full_alias` (`L90`) | **400** | `2 consecutive dot signs aren't allowed in an email address` (`L92`) | — |
| — | **success** | **201** | `jsonify(alias=full_alias, **serialize_alias_info_v2(get_alias_info_v2(alias)))` (`L109-112`) | — |

### 2.4 Full rejection table — `POST /api/v3/alias/custom/new`

v3 accepts a list of mailboxes and adds several checks that **all run *before* the suffix
signature check**. Evidence: `app/api/views/new_custom_alias.py`.

| # | Condition (code) | Status | Literal `error` body | Log call |
|---|------------------|--------|----------------------|----------|
| 1 | `not user.can_create_new_alias()` (`L137`) | **400** | `You have reached the limitation of a free account with the maximum of {MAX_NB_EMAIL_FREE_PLAN} aliases, please upgrade your plan to create more aliases` (`L140-142`) | `LOG.d("user %s cannot create any custom alias", user)` (`L138`) |
| 2 | empty body — `if not data` (`L150`) | **400** | `request body cannot be empty` (`L151`) | — |
| 3 | `not isinstance(data, dict)` (`L153`) | **400** | `request body does not follow the required format` (`L154`) | — |
| 4 | `not check_alias_prefix(alias_prefix)` (`L167`) | **400** | `alias prefix invalid format or too long` (`L168`) | — |
| 5 | `not isinstance(mailbox_ids, list)` (`L171`) | **400** | `mailbox_ids must be an array of id` (`L172`) | — |
| 6 | bad / foreign / unverified mailbox (`L174-176`) | **400** | `Errors with Mailbox` (`L177`) | — |
| 7 | `not mailboxes` — none selected (`L180`) | **400** | `At least one mailbox must be selected` (`L181`) | — |
| 8 | `check_suffix_signature()` → `None` (`L185-186`) | **412** | `Alias creation time is expired, please retry` (`L188`) | `LOG.w("Alias creation time expired for %s", user)` (`L187`) |
| 9 | `except Exception` (rare / unreachable, see §2.2) (`L189`) | **400** | `Tampered suffix` (`L191`) | `LOG.w("Alias suffix is tampered, user %s", user)` (`L190`) |
| 10 | `not verify_prefix_suffix(...)` (`L193`) | **400** | `wrong alias prefix or suffix` (`L194`) | `LOG.e(...)` inside `verify_prefix_suffix` |
| 11 | alias collision (`L197-201`) | **409** | `alias {full_alias} already exists` (`L203`) | `LOG.d("full alias already used %s", full_alias)` (`L202`) |
| 12 | `".." in full_alias` (`L205`) | **400** | `2 consecutive dot signs aren't allowed in an email address` (`L207`) | — |
| — | **success** | **201** | alias payload (`L232-235`) | — |

**`check_alias_prefix()`** [`app/alias_utils.py:L418-425`] returns `False` if
`len(alias_prefix) > 40` [`L419-420`] or if the prefix fails the regex
`_ALIAS_PREFIX_PATTERN = r"[0-9a-z-_.]{1,}"` [`L415`, `L422-423`] — i.e. only lowercase
letters, digits, `.`, `-`, and `_`, up to 40 characters.

### 2.5 Web (dashboard) path — important contrast

The numeric status codes above apply to the **API** endpoints. The web UI path
`app/dashboard/views/custom_alias.py` shares the **same verifier** (it imports
`get_alias_suffixes`, `check_suffix_signature`, `verify_prefix_suffix`
[`app/dashboard/views/custom_alias.py:L7-11`]) and the **same concurrency lock**
(`@parallel_limiter.lock(name="alias_creation")` [`L33`]), but it responds with **`flash()`
messages + `redirect()` (HTTP 302)** rather than JSON status codes:

| Stage | Behavior | Lines |
|-------|----------|-------|
| Quota gate | `LOG.d("%s can't create new alias", current_user)` + flash *"You have reached free plan limit, please upgrade to create new aliases"* | `L36-37` |
| Expired/invalid suffix | `LOG.w("Alias creation time expired for %s", current_user)` + flash *"Alias creation time is expired, please retry"* | `L92-93` |
| `except Exception` | `LOG.w("Alias suffix is tampered, user %s", current_user)` + flash *"Unknown error, refresh the page"* | `L96-97` |

So if the reported failures originate from the **web dashboard**, expect a redirect with a
flashed warning, not a 4xx JSON body — while the underlying suffix logic (and thus the same
expired‑vs‑tampered conflation) is identical.


---

## 3. Q2 — Server-Console Validation Log Entries

All log lines come from a single custom logger named **`SL`** that writes to **stdout**
[`app/log.py:L79` (`LOG = _get_logger("SL")`); `L41` (`logging.StreamHandler(sys.stdout)`)].
Werkzeug's own access logs are disabled [`app/log.py:L70-71`], and timestamps are emitted in
**UTC** [`app/log.py:L43` (`converter = time.gmtime`)].

### 3.1 The exact console format

```text
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

(verbatim from `app/log.py:L12-15`). A real line captured from the running app — note the empty
`message_id` rendered as ` -  - `:

```text
2026-06-26 20:51:28,141 - SL - DEBUG - 1185 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

### 3.2 Level shortcuts — and the `LOG.e` nuance

The logger is monkey‑patched with one‑letter shortcuts [`app/log.py:L74-77`]:

| Shortcut | Maps to | Level | Note |
|----------|---------|-------|------|
| `LOG.d` | `logging.Logger.debug` | DEBUG | |
| `LOG.i` | `logging.Logger.info` | INFO | |
| `LOG.w` | `logging.Logger.warning` | WARNING | |
| `LOG.e` | `logging.Logger.exception` | **ERROR** | **logs at ERROR level *with a stack traceback*** |

> ⚠️ **`LOG.e` is `logging.Logger.exception`, not `error`.** Every `LOG.e(...)` call prints an
> **ERROR line followed by a traceback** — even when no exception was actually raised. This is a
> notable log artifact on the suffix path (see the `verify_prefix_suffix` entries below).

### 3.3 Validation-relevant log entries on the creation path

The following entries are emitted during signed‑suffix verification and the surrounding
validation, with their level and source line:

- **WARNING — 412 "expired" path:**
  `LOG.w("Alias creation time expired for %s", user)`
  [`new_custom_alias.py:L72` (v2), `L187` (v3)]. **Because of the
  [KEY FINDING](#22-key-finding--expired-and-tampered-suffixes-both-return-412), this same warning is
  written for a tampered/malformed suffix, not only for a genuinely expired one.**
- **WARNING — rare/unreachable 400 "Tampered suffix" path:**
  `LOG.w("Alias suffix is tampered, user %s", user)`
  [`new_custom_alias.py:L75` (v2), `L190` (v3)]. In practice this line will seldom appear, since
  ordinary tampering is caught earlier and reported as the 412 above.
- **ERROR *with traceback* — inside `verify_prefix_suffix()`** (accompanies the 400
  "wrong alias prefix or suffix" response):
  - `LOG.e("wrong alias suffix %s, user %s", alias_suffix, user)`
    [`app/alias_suffix.py:L61, L84, L88`]
  - `LOG.e("User %s submits a wrong alias suffix %s", user, alias_suffix)`
    [`app/alias_suffix.py:L78`]

  These print a **traceback even though no real exception occurred** (a direct consequence of
  `LOG.e == logging.Logger.exception`) — a potentially confusing artifact when grepping logs
  during a diagnosis.
- **DEBUG — quota‑gate 400 path:**
  `LOG.d("user %s cannot create any custom alias", user)`
  [`new_custom_alias.py:L49` (v2), `L138` (v3)].
- **DEBUG — 409 collision path:**
  `LOG.d("full alias already used %s", full_alias)`
  [`new_custom_alias.py:L87` (v2), `L202` (v3)].
- **WARNING — global 429 handler** (fires for *any* rate‑limit breach, all three mechanisms):
  `LOG.w("Client hit rate limit on path %s, user:%s", request.path, get_current_user())`
  [`server.py:L364-368`].
- **INFO — bucket‑quota breach only** (see [§5](#5-q4q5--success-path-quota-checks--what-is-logged)):
  `LOG.i("Rate limit hit for {lock_name} (bucket id {bucket_id}) -> {value}/{max_hits}")`
  [`app/rate_limiter.py:L33-35`].

> **DEBUG visibility.** The `SL` logger level is set to `DEBUG` [`app/log.py:L51`], so DEBUG
> lines *are* emitted to stdout by default in this codebase. If a deployment filters DEBUG at
> the aggregator, the quota‑gate (item 4) and collision (item 5) lines may not be visible even
> though they were logged.


---

## 4. Q3 — Rate-Limit Headers & the Three Limiting Mechanisms

### 4.1 Rate-limit headers — ABSENT

> **Answer to Q3: No rate‑limit headers appear on responses.** There are **no**
> `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, or `Retry-After` headers.

Three independent pieces of evidence support this:

1. **The limiter is constructed without header support.**
   `limiter = Limiter(key_func=__key_func)` [`app/extensions.py:L23`] — there is **no**
   `headers_enabled` argument. In Flask‑Limiter 1.4 the `X-RateLimit-*` and `Retry-After`
   headers are written **only when header support is explicitly enabled**, via
   `headers_enabled=True` on the constructor or the `RATELIMIT_HEADERS_ENABLED` config flag.
2. **No header‑enabling config exists anywhere.** A repository‑wide search for
   `RATELIMIT_HEADERS_ENABLED` and `headers_enabled` returns **nothing** (outside of tests).
3. **The global 429 handler discards any limiter‑set headers.** Even on a breach, the handler
   returns a *fresh* response object:

   ```python
   # server.py:L362-372
   @app.errorhandler(429)
   def rate_limited(e):
       LOG.w("Client hit rate limit on path %s, user:%s", request.path, get_current_user())
       if request.path.startswith("/api/"):
           return jsonify(error="Rate limit exceeded"), 429
       else:
           return render_template("error/429.html"), 429
   ```

   Because a brand‑new `jsonify(...)` response is returned, any headers the limiter might have
   attached to the original error are not propagated.

**Live confirmation (read‑only, throwaway).** A forced Flask‑Limiter breach on
`POST /api/v3/alias/custom/new` returned `429` with body `{"error": "Rate limit exceeded"}` and
a header set containing **only**:

```text
['Access-Control-Allow-Origin', 'Content-Length', 'Content-Type', 'Set-Cookie']
```

No `X-RateLimit-*` and no `Retry-After`. Standard headers such as
`Content-Type: application/json` are present; rate‑limit‑specific headers are not.

### 4.2 Three INDEPENDENT limiting mechanisms — do **not** conflate

This is critical for diagnosing the *intermittent* failures: **three different mechanisms can
each reject a creation attempt, all surfacing as HTTP 429** (for API paths) with the **same**
`{"error":"Rate limit exceeded"}` body via the same global handler — but for completely
different reasons.

#### Mechanism ① — Flask-Limiter HTTP route limit

- Applied by `@limiter.limit(ALIAS_LIMIT)` on each endpoint.
- `ALIAS_LIMIT` default = `"100/day;50/hour;5/minute"` [`app/config.py:L448`].
- Keyed per user (`userid:{id}`) when authenticated, else per IP (`ip:{addr}`)
  [`app/extensions.py:L14-19`].
- A breach is raised by Flask‑Limiter as HTTP 429 → routed to the global handler
  ([§4.1](#41-rate-limit-headers--absent)).
- **Disabled entirely** when `config.DISABLE_RATE_LIMIT` is truthy
  [`app/extensions.py:L26-28`], where
  `DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ` [`app/config.py:L602`].

#### Mechanism ② — Bucket-based alias-creation quota

- `rate_limiter.check_bucket_limit()` is invoked inside `Alias.create()`
  ([§5.2](#52-bucket-quota-inside-aliascreate)).
- Defaults: **Free** `[(10,900),(50,3600)]` and **Paid** `[(50,900),(200,3600)]`
  (i.e. *hits, seconds*) [`app/config.py:L554-559`].
- Backed by a Redis `INCR` on a wall‑clock‑aligned bucket; raises
  `werkzeug.exceptions.TooManyRequests` (429) when a bucket is exceeded
  [`app/rate_limiter.py:L31-40`].

#### Mechanism ③ — Concurrency lock (current-user-or-IP scoped)

- Applied by `@parallel_limiter.lock(name="alias_creation")`.
- `acquire_lock` does a Redis `SET ... nx=True, ex=timedelta(seconds=5)`
  [`app/parallel_limiter.py:L30-32`]; if the key already exists (a concurrent in‑flight create
  sharing the same lock key), it raises `exceptions.TooManyRequests()` → 429
  [`app/parallel_limiter.py:L34`].
- Lock key is selected per caller [`app/parallel_limiter.py:L55-58`]: when `current_user`
  exposes an `id` (Flask‑Login / session auth) the key is `cl:{current_user.id}:alias_creation`;
  **otherwise it falls back to `cl:{request.remote_addr}:alias_creation`** (e.g. API‑key‑only
  requests, where `current_user` is the anonymous user and has no `id` —
  see [`app/api/base.py:L34`], which sets `g.user` but never calls `login_user()`). TTL is
  **5 seconds** (`max_wait_secs=5` [`L23`, `L70`]).

> **Diagnostic takeaway.** Because all three produce an identical `429
> {"error":"Rate limit exceeded"}` body and identical (absent) headers, the *only* way to tell
> them apart from the outside is **timing and rate**: ① trips at the per‑minute/hour/day request
> rate, ② trips at the per‑15‑minute/per‑hour *creation* counts, and ③ trips only on
> *simultaneous* requests within a 5‑second window. The global‑handler WARNING line
> ([§3.3](#33-validation-relevant-log-entries-on-the-creation-path)) is the same for all three;
> only mechanism ② additionally logs an INFO line naming the bucket
> ([§5.2](#52-bucket-quota-inside-aliascreate)).


---

## 5. Q4/Q5 — Success-Path Quota Checks & What Is Logged

On a **successful** creation, **two** quota‑related checks run: a gate
(`User.can_create_new_alias()`) at the top of the endpoint, and the **bucket quota** inside
`Alias.create()`.

### 5.1 `User.can_create_new_alias()` (the gate)

```python
# app/models.py:L867-884
def can_create_new_alias(self) -> bool:
    """
    Whether user can create a new alias. User can't create a new alias if
    - has more than 15 aliases in the free plan, *even in the free trial*
    """
    if not self.is_active():                 # L872-873
        return False
    if self.disabled:                        # L875-876
        return False
    if self.lifetime_or_active_subscription():   # L878-879
        return True
    else:
        return (                             # L881-884
            Alias.filter_by(user_id=self.id).count()
            < self.max_alias_for_free_account()
        )
```

The free‑plan ceiling comes from `max_alias_for_free_account()` [`app/models.py:L858-865`]:

```python
def max_alias_for_free_account(self) -> int:
    if self.FLAG_FREE_OLD_ALIAS_LIMIT == self.flags & self.FLAG_FREE_OLD_ALIAS_LIMIT:
        return config.MAX_NB_EMAIL_OLD_FREE_PLAN   # default 15
    else:
        return config.MAX_NB_EMAIL_FREE_PLAN       # default 5
```

- `FLAG_FREE_OLD_ALIAS_LIMIT = 1 << 2` [`app/models.py:L341`].
- `MAX_NB_EMAIL_OLD_FREE_PLAN` default **15**, `MAX_NB_EMAIL_FREE_PLAN` default **5**
  [`app/config.py:L121-124, L126`]. So a normal free account is limited to **5** aliases;
  legacy accounts carrying the `FLAG_FREE_OLD_ALIAS_LIMIT` bit get **15**.

**Logging:** this method **logs nothing itself**. A log line appears only on the *failure* path,
from the *caller*: `LOG.d("user %s cannot create any custom alias", user)`
[`new_custom_alias.py:L49` (v2), `L138` (v3)].

> **FINDING (documented, not fixed).** The docstring at `app/models.py:L870` says
> *"has more than 15 aliases in the free plan"* — this is **stale** relative to the configurable
> default of **5** (`MAX_NB_EMAIL_FREE_PLAN`). It could mislead anyone reading the code while
> diagnosing a quota‑gate rejection. Per the read‑only mandate, this is reported only.

### 5.2 Bucket quota inside `Alias.create()`

```python
# app/models.py:L1627-1641
@classmethod
def create(cls, **kw):
    ...
    new_alias = cls(**kw)
    user = User.get(new_alias.user_id)
    if user.is_premium():
        limits = config.ALIAS_CREATE_RATE_LIMIT_PAID    # [(50,900),(200,3600)]
    else:
        limits = config.ALIAS_CREATE_RATE_LIMIT_FREE    # [(10,900),(50,3600)]
    # limits is array of (hits,days)        # <-- comment is misleading; values are SECONDS
    for limit in limits:
        key = f"alias_create_{limit[1]}d:{user.id}"
        rate_limiter.check_bucket_limit(key, limit[0], limit[1])
```

```python
# app/rate_limiter.py:L19-42
def check_bucket_limit(lock_name=None, max_hits=5, bucket_seconds=3600):
    int_time = int(datetime.utcnow().timestamp())               # L25
    bucket_id = int_time - (int_time % bucket_seconds)          # L26  (wall-clock window)
    bucket_lock_name = f"bl:{lock_name}:{bucket_id}"            # L27
    if not lock_redis:                                          # L28-29  NO-OP w/o Redis
        return
    try:
        value = lock_redis.incr(bucket_lock_name, bucket_seconds)   # L31  Redis INCR
        if value > max_hits:                                        # L32
            LOG.i(f"Rate limit hit for {lock_name} (bucket id {bucket_id}) -> {value}/{max_hits}")  # L33-35
            newrelic.agent.record_custom_event(                     # L36-39
                "BucketRateLimit",
                {"lock_name": lock_name, "bucket_seconds": bucket_seconds},
            )
            raise werkzeug.exceptions.TooManyRequests()             # L40
    except (redis.exceptions.RedisError, AttributeError):           # L41-42
        LOG.e("Cannot connect to redis")
```

Behavior on a **successful, under‑limit** creation:

- The bucket key is `bl:alias_create_{seconds}d:{user.id}:{bucket_id}` — e.g. for the first
  free limit, `lock_name = "alias_create_900d:{user.id}"` [`app/models.py:L1640`].
- It performs **only a Redis `INCR`** [`app/rate_limiter.py:L31`] and returns — **it logs
  nothing**. The `LOG.i(...)` line, the New Relic `BucketRateLimit` event, and the 429 are all
  inside the `if value > max_hits:` branch [`L32-40`], which does not execute under the limit.
- If Redis is not initialized, the whole check is a **silent no‑op** (`if not lock_redis:
  return` [`L28-29`]) — see [§6](#6-root-cause-analysis-of-the-intermittent-failures).

> Two further notes on the code (documented as findings, not fixed): the loop comment
> *"# limits is array of (hits,days)"* [`app/models.py:L1638`] and the `…d` suffix in the key
> [`L1640`] both say *days*, but the values are **seconds** (`900` = 15 min, `3600` = 1 hour).

### 5.3 Q5 — what is logged on a successful creation: **nothing**

> **Answer to Q5.** When the system verifies whether the user may create another alias, on a
> **successful, under‑quota** creation the quota machinery logs **no values at all**:
>
> - `can_create_new_alias()` logs nothing on success (only the *caller* logs, and only on the
>   *failure* path).
> - `check_bucket_limit()` performs only a Redis `INCR` and logs nothing unless a bucket is
>   exceeded.
>
> The values `{value}/{max_hits}`, the `bucket_id`, and the `lock_name` (e.g.
> `alias_create_900d:{user.id}`) are logged **only when a limit is breached**
> [`app/rate_limiter.py:L33-35`] — **never** on a successful creation.

### 5.4 Q4 — summary of the success-path checks

| Order | Check | Where | Effect on success | Logs on success? |
|-------|-------|-------|-------------------|------------------|
| 1 | `user.can_create_new_alias()` | endpoint (`new_custom_alias.py:L48/L137`) | passes (active, not disabled, under limit or subscribed) | No |
| 2 | `check_bucket_limit()` × N windows | `Alias.create()` (`models.py:L1639-1641`) | Redis `INCR` per window, under `max_hits` | No |

Both pass silently; the alias row is then written via `Alias.create()` and `Session.commit()`,
and the endpoint returns **201** with the serialized alias payload.


---

## 6. Root-Cause Analysis of the Intermittent Failures

The reported symptom — *intermittent* validation failures that *don't match the expected
behavior* — is best explained by **four independent, time/state‑dependent triggers**. Each can
fire (or not) for identical request inputs depending on timing, concurrency, and environment.

### 6.1 The 600-second suffix expiry (and the expired/tampered conflation)

The signed suffix is valid for only **600 s** after the *options* endpoint issued it
[`app/alias_suffix.py:L40`]. Any of the following can push the submission past that boundary
and produce **412 "Alias creation time is expired, please retry"**:

- slow form completion by the user,
- a stale or cached `/alias/options` response reused after some delay,
- a delayed retry of a previously‑fetched suffix.

Crucially, because of the [KEY FINDING](#22-key-finding--expired-and-tampered-suffixes-both-return-412), a
**tampered or malformed** suffix yields the **same** 412 "expired" response. A client that
mangles the `signed_suffix` (truncation, re‑encoding, a stale value from a different secret,
copy/paste corruption) will be told the suffix *expired* — so the failure looks like a timing
problem when it may actually be a data‑integrity problem. This is the most likely source of
*"failures that don't match the expected behavior."*

### 6.2 Bucket-quota wall-clock windows

The bucket quota resets on **fixed wall‑clock boundaries**:
`bucket_id = int_time - (int_time % bucket_seconds)` [`app/rate_limiter.py:L25-26`]. The counter
is per *aligned* window, not a rolling window. Consequently **the same number of requests can
pass or fail depending on where "now" falls within the window**: a burst that straddles a window
boundary is split across two buckets and succeeds, while the identical burst landing entirely
inside one window trips the limit. With Free `[(10,900),(50,3600)]` / Paid
`[(50,900),(200,3600)]`, this yields intermittent 429s that correlate with the clock, not with
the request payload.

### 6.3 Concurrency lock — current-user-or-IP (5-second window)

A second *simultaneous* create sharing the same lock key — while a prior create is still in
flight — fails to acquire the Redis lock and is rejected with **429**
[`app/parallel_limiter.py:L30-34`]. The lock key is `cl:{current_user.id}:alias_creation` for
Flask‑Login / session‑authenticated users (who expose an `id`), and **falls back to
`cl:{request.remote_addr}:alias_creation`** otherwise — e.g. API‑key‑only requests, where
`current_user` is anonymous [`app/parallel_limiter.py:L55-58`]. The lock has a **5‑second TTL**.
This depends entirely on **request overlap/timing**: sequential requests are fine; two requests
racing within ~5 s (e.g. a double‑click, a retrying client, or parallel automation) — and, under
the IP‑keyed fallback, even *different* API‑key users behind the same IP/NAT — intermittently
collide.

### 6.4 Redis availability / environment sensitivity

This is the classic *"works on my machine / fails in prod"* driver. **When Redis is not
initialized, BOTH the bucket quota and the concurrency lock silently no‑op:**

- bucket quota: `if not lock_redis: return` [`app/rate_limiter.py:L28-29`];
- concurrency lock: `if not lock_redis: return f(*args, **kwargs)`
  [`app/parallel_limiter.py:L51-52`].

Redis is wired by `initialize_redis_services()` [`app/redis_services.py:L9-25`]. So **identical
inputs behave differently across environments**: in a Redis‑backed deployment both ② and ③ are
active; in one without Redis, they vanish. The Flask‑Limiter HTTP limit ① has its own storage
configuration, independent of these two.

Layer on `DISABLE_RATE_LIMIT` [`app/config.py:L602`]: if the environment variable is present in
one environment but not another, **mechanism ① disappears entirely** in the first. Differences in
Redis presence and `DISABLE_RATE_LIMIT` between dev/staging/prod are a prime cause of
intermittency that tracks the *environment*, not the request.

### 6.5 Putting it together — a diagnostic checklist

| Symptom observed | Most likely mechanism | Distinguishing evidence |
|------------------|-----------------------|--------------------------|
| `412 "…expired, please retry"` for a *fresh* suffix | Expired/tampered conflation ([§2.2](#22-key-finding--expired-and-tampered-suffixes-both-return-412)) | Verify the `signed_suffix` is byte‑exact and < 600 s old; a corrupted value also returns 412 |
| `429 "Rate limit exceeded"`, sporadic, tracks the clock | Bucket quota ② ([§5.2](#52-bucket-quota-inside-aliascreate)) | INFO log `Rate limit hit for alias_create_…` names the bucket |
| `429`, only under bursts of requests | Flask‑Limiter ① ([§4.2](#42-three-independent-limiting-mechanisms--do-not-conflate)) | Trips at 5/minute; disabled by `DISABLE_RATE_LIMIT` |
| `429`, only on *simultaneous* requests | Concurrency lock ③ ([§4.2](#42-three-independent-limiting-mechanisms--do-not-conflate)) | Only on overlap within 5 s sharing the lock key (`cl:{user.id}`, or `cl:{remote_addr}` for API‑key‑only calls); no per‑request INFO log |
| Behavior differs between environments | Redis absence / `DISABLE_RATE_LIMIT` ([§6.4](#64-redis-availability--environment-sensitivity)) | ② and ③ no‑op without Redis; ① off when `DISABLE_RATE_LIMIT` set |


---

## Appendix A — Methodology & Reproduction (temporary, never committed)

The behaviors above were verified by reading the source and by **reproducing them live** against
the running application. The commands/scripts below are **throwaway** — they were used for
observation only and were **not** added to the repository. (The findings in this document are
the only artifact produced.)

**Authentication.** API requests carry the API key in the **`Authentication`** HTTP header
[`app/api/base.py:L17`]; mirrors `tests/api/test_alias_options.py:L13-15`
(`headers={"Authentication": api_key.code}`). Session‑cookie auth also works via the
dashboard/login flow used by `tests/utils.py:login()`.

**Fetch a fresh signed suffix.** `GET /api/v4/alias/options` (or `/api/v5/alias/options`) with
the `Authentication` header → the `suffixes` field contains `[suffix, signed_suffix]` pairs (v4)
or objects (v5). Use a `signed_suffix` value
[`app/api/views/alias_options.py:L69,L72,L140-150`].

**Create (v2).** `POST /api/v2/alias/custom/new` with JSON
`{"alias_prefix": "...", "signed_suffix": "..."}` → `201` on success (pattern:
`tests/api/test_new_custom_alias.py:L13-29`).

**Create (v3).** Add `"mailbox_ids": [<default_mailbox_id>]` (pattern:
`tests/api/test_new_custom_alias.py:L45-62`).

**Forge a fresh/known suffix in a script** (as the tests do):

```python
from app.alias_suffix import signer
signed = signer.sign(".word@domain").decode()   # tests: L18 / L50 / L266
```

**Reproduce the suffix expiry / the KEY FINDING.**

- Wait > 600 s after issuing a suffix, then POST → `412 {"error":"Alias creation time is
  expired, please retry"}`.
- To confirm the conflation, POST a *deliberately corrupted* `signed_suffix` and observe the
  **same** 412 (not 400). Observed live:

  ```text
  VALID    status= 201
  TAMPERED status= 412 body= {'error': 'Alias creation time is expired, please retry'}
  GARBAGE  status= 412 body= {'error': 'Alias creation time is expired, please retry'}
  ```

- The exception hierarchy can be confirmed directly:

  ```python
  import itsdangerous as i
  i.__version__                                  # '1.1.0'
  issubclass(i.SignatureExpired, i.BadSignature)     # True
  issubclass(i.BadTimeSignature, i.BadSignature)     # True
  ```

**Reproduce the Flask‑Limiter HTTP 429.** Set `config.DISABLE_RATE_LIMIT = False`, then loop
more than 5 POSTs/minute; expect the final `status_code == 429` and body
`{"error":"Rate limit exceeded"}` (pattern: `tests/api/test_new_custom_alias.py:L255-283`,
including the flask‑limiter unit‑test workaround `g._rate_limiting_complete = False` at `L279`).
Observed header set on the 429: `Access-Control-Allow-Origin, Content-Length, Content-Type,
Set-Cookie` — **no** `X-RateLimit-*` / `Retry-After`.

**Reproduce the bucket quota / concurrency lock.** Requires Redis. Exceed the Free/Paid window
counts to trip the bucket quota, or fire two simultaneous creates within 5 s to trip the lock.
Toggle Redis availability to demonstrate the silent no‑op behavior
([§6.4](#64-redis-availability--environment-sensitivity)).

**Environment prerequisites.** Python 3.10, PostgreSQL [`example.env` `DB_URI`], and Redis
[`app/redis_services.py`]; the runtime pins are listed in [Appendix C](#appendix-c--pinned-dependency-versions).
For the live runs in this document, requests were issued through the Flask test client with
`CONFIG` pointed at `tests/test.env` (`MEM_STORE_URI=redis://localhost`).

---

## Appendix B — Citations / Evidence

Every claim in this document is backed by the following source locations (key line ranges used):

- `app/alias_suffix.py` — L11 (signer), L37‑42 (`check_suffix_signature`), L45‑91
  (`verify_prefix_suffix`), L94‑192 (`get_alias_suffixes`)
- `app/api/views/new_custom_alias.py` — L28‑112 (`new_custom_alias_v2`), L115‑235
  (`new_custom_alias_v3`)
- `app/api/views/alias_options.py` — L69, L72 (v4 issuance), L140, L143‑150 (v5 issuance)
- `app/api/base.py` — L11 (blueprint `url_prefix="/api"`), L16‑43 (`authorize_request`,
  `Authentication` header at L17), L52‑60 (`require_api_auth`)
- `app/alias_utils.py` — L414‑425 (`check_alias_prefix`, `_ALIAS_PREFIX_PATTERN` at L415)
- `app/extensions.py` — L14‑19 (`__key_func`), L23 (`Limiter` — no `headers_enabled`), L26‑28
  (`DISABLE_RATE_LIMIT` request filter)
- `app/parallel_limiter.py` — L19‑73 (lock; `acquire_lock` L30‑34, no‑op L51‑52, key‑selection L55‑58 (user‑id or IP fallback),
  5 s TTL L23/L70)
- `app/rate_limiter.py` — L19‑42 (`check_bucket_limit`; `bucket_id` L25‑26, no‑op L28‑29,
  `INCR` L31, `LOG.i` L33‑35, New Relic L36‑39, 429 L40, `LOG.e` L41‑42)
- `app/models.py` — L341 (`FLAG_FREE_OLD_ALIAS_LIMIT`), L858‑865 (`max_alias_for_free_account`),
  L867‑884 (`can_create_new_alias`; stale docstring L870), L1627‑1641 (`Alias.create` bucket
  quota; misleading comment L1638, key L1640)
- `app/config.py` — L121‑124 (`MAX_NB_EMAIL_FREE_PLAN`=5 default), L126
  (`MAX_NB_EMAIL_OLD_FREE_PLAN`=15), L201 (`CUSTOM_ALIAS_SECRET`), L448 (`ALIAS_LIMIT`),
  L554‑559 (`ALIAS_CREATE_RATE_LIMIT_FREE`/`_PAID`), L602 (`DISABLE_RATE_LIMIT`)
- `server.py` — L167 (`limiter.init_app`), L362‑372 (global 429 handler; WARNING L364‑368,
  API 429 body L370)
- `app/log.py` — L12‑15 (format string), L40‑45 (stdout `StreamHandler`, UTC converter L43),
  L70‑71 (werkzeug disabled), L74‑77 (`d/i/w/e` shortcuts; `e`=`exception`), L79
  (`LOG = _get_logger("SL")`)
- `app/redis_services.py` — L9‑25 (`initialize_redis_services`)
- `app/dashboard/views/custom_alias.py` — L7‑11 (shared verifier imports), L33 (lock), L36‑37
  (quota gate + `LOG.d`), L92‑93 (expired flash), L96‑97 (tampered flash)
- `tests/api/test_new_custom_alias.py` — L4 (`signer` import), L13‑62 (v2/v3 create patterns),
  L255‑283 (429 reproduction; body assertion L283)
- `tests/api/test_alias_options.py` — L9‑20 (`Authentication` header pattern)
- `tests/dashboard/test_custom_alias.py` — L6‑11 (shared verifier imports incl. `signer`), L28‑54 (`signer.sign` create pattern), L367‑395 (dashboard rate‑limit/429 behavior; 429 + body assertions L394‑395)

---

## Appendix C — Pinned Dependency Versions

From `poetry.lock` (confirmed at runtime in the project's Python 3.10 environment):

| Package | Version |
|---------|---------|
| flask | 1.1.2 |
| flask-limiter | 1.4 |
| itsdangerous | 1.1.0 |
| werkzeug | 1.0.1 |
| redis | 4.6.0 |
| limits | 1.5.1 |
| sqlalchemy | 1.3.24 |
| newrelic | 8.8.0 |

The **`itsdangerous` 1.1.0** exception hierarchy
(`SignatureExpired ⊂ BadTimeSignature ⊂ BadSignature`) is precisely what makes the 412/400
conflation in [§2.2](#22-key-finding--expired-and-tampered-suffixes-both-return-412) true; a different
`itsdangerous` major version could change that relationship and therefore the behavior.

---

### Summary of findings reported (not fixed — read-only task)

1. **412‑vs‑400 conflation:** expired and tampered/malformed suffixes both return
   `412 "Alias creation time is expired, please retry"`; the `400 "Tampered suffix"` branch is
   effectively unreachable ([§2.2](#22-key-finding--expired-and-tampered-suffixes-both-return-412)).
2. **Stale docstring:** `can_create_new_alias()` says "more than 15 aliases" while the default
   free limit is 5 ([§5.1](#51-usercan_create_new_alias-the-gate)).
3. **Misleading units:** `Alias.create()`'s `# limits is array of (hits,days)` comment and the
   `…d` key suffix denote *seconds*, not days
   ([§5.2](#52-bucket-quota-inside-aliascreate)).
4. **No rate‑limit headers:** Flask‑Limiter header support is not enabled, and the global 429
   handler returns a fresh body, so no `X-RateLimit-*`/`Retry-After` headers are emitted
   ([§4.1](#41-rate-limit-headers--absent)).

