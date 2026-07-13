# SimpleLogin — What Happens When You Create a New Alias (Branch app_2cd6ee777f8c)

This document answers six questions about SimpleLogin's **"create a new alias"** flow from **observed runtime evidence** captured while the application was actually running in its **default, out-of-the-box development configuration**. Every behavioral claim is paired with (a) the actual captured output (real log lines, full HTTP request/response blocks, SQL statements, error text) and (b) a `file:line` citation naming the specific function/method that performs the work.

Labelling conventions used throughout:

- **OBSERVED** — the value was produced at runtime by exercising the real web entry point and captured directly. Unless a block is marked otherwise, it is OBSERVED in the default configuration.
- **INFERRED** — the statement is derived from reading the source code, not from a captured runtime observation. Every such statement is explicitly labelled.
- **NON-CANONICAL** — the observation was produced under a non-default configuration (e.g., a Redis store wired up by hand). Default-configuration values are always reported first; any non-default observation is explicitly labelled.

The six questions answered:

- **Q1 — Run and observe** (Section 2): bring the app up, log in, create an alias, watch the flow live.
- **Q2 — Frontend request** (Section 3): the exact HTTP method, URL, content type, body fields, and CSRF handling.
- **Q3 — Backend response** (Section 4): status code(s), payload, headers, redirect location, flash messaging.
- **Q4 — Database changes** (Section 5): which records are inserted/updated, how many tables, related entities.
- **Q5 — Logs & background work** (Section 6): background tasks, follow-up events, additional work beyond the initial write.
- **Q6 — Error paths** (Section 7): how validation, database, and network/service failures surface in the response, logs, and DB state.

> **The single most important architectural finding:** in the default development configuration, creating a random alias writes **3 tables** (`alias`, `daily_metric`, `alias_audit_log`), the domain event is a **logged no-op** (no `sync_event`, no `NOTIFY`), and the HTTP response is a **Post/Redirect/Get 302 + flash message**. A custom alias with a secondary mailbox writes **4 tables** (adds `alias_mailbox`). This canonical default behavior is cleanly separated from configuration-dependent behavior (a partner/webhook setup would add a `sync_event` INSERT + `NOTIFY`). The exact same creation core — `Alias.create` (`app/models.py:1627-1692`) — backs both the web form and the REST API; only the HTTP envelope differs (302 + flash vs. 201 JSON).

---

## Section 1 — Overview: Environment, Versions, and Bootstrap Commands

All facts in this section are **OBSERVED** from the running instance.

### 1.1 Repository and runtime

| Item | Value |
|------|-------|
| Repo path | `/tmp/blitzy/app/app_2cd6ee777f8c_36621f` |
| Git branch | `app_2cd6ee777f8c` |
| HEAD commit | `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` |
| Python | **3.10.20** in an in-project venv `./.venv` |
| Dev web server | **Werkzeug/1.0.1**, port **7777** (`server.py:588` — `app.run(debug=True, port=7777)`) |
| PostgreSQL | **16.14** on `:5432` |
| Redis | **7.0.15** on `:6379` (present but *not* wired into rate limiting by default — see §7 B4) |

The project targets Python `^3.10` (`pyproject.toml:61`, `CONTRIBUTING.md`). **OBSERVED rationale:** the container ships Python 3.12, which cannot build the locked C-extensions (`frozenlist` / `cbor2` / `pyre2`), so Python **3.10.20** was used. The alias-creation logic is version-independent, so this does not affect any observed value.

### 1.2 Key pinned dependencies (from `poetry.lock`)

| Package | Version | Role in the alias-creation flow |
|---------|---------|--------------------------------|
| flask | 1.1.2 | routing, request/response, flash messaging |
| flask-login | 0.5.0 | `@login_required`, `current_user` |
| flask-wtf | 0.14.3 | CSRF-protected forms (`CSRFValidationForm`) |
| wtforms | 2.3.3 | form field definitions |
| flask-limiter | 1.4 | route-level rate limiting (`ALIAS_LIMIT`) |
| sqlalchemy | 1.3.24 | ORM, scoped `Session`, SQL emission |
| alembic | 1.4.3 | schema migrations (`alembic upgrade head`) |
| psycopg2-binary | 2.9.3 | PostgreSQL driver |
| redis | 4.6.0 | client backing the per-user token bucket + Redlock |
| protobuf | 5.27.1 | encoding of the `AliasCreated` domain event |
| arrow | 0.16.0 | timestamps on models |

### 1.3 Canonical bootstrap (default configuration)

The application was brought up exactly as the contributor guide prescribes — `CONTRIBUTING.md:106` gives the one-liner `alembic upgrade head && flask dummy-data && python3 server.py`, and `CONTRIBUTING.md:109` gives the login instruction (open `http://localhost:7777`, log in with `john@wick.com / password`). The exact command block used:

```
cp example.env .env         # .env is gitignored; == example.env verbatim.
                            # NO MEM_STORE_URI / REDIS_URL / EVENT_WEBHOOK / DISABLE_RATE_LIMIT set.
                            # DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
                            # URL=http://localhost:7777 ; EMAIL_DOMAIN=sl.local ; FLASK_SECRET=secret
alembic upgrade head        # EXIT 0 -> 77 tables; head revision 32f25cbf12f6 (alias_audit_log_index_created_at)
FLASK_APP=wsgi.py flask dummy-data   # server.py:490-497 -> fake_data()+add_sl_domains()+add_proton_partner()
python3 server.py           # dev server :7777 ; startup logs ">>> URL: http://localhost:7777",
                            # "MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value"
```

The `dummy-data` CLI command is defined at `server.py:490-497` and runs `fake_data()`, `add_sl_domains()`, and `add_proton_partner()`:

```python
    @app.cli.command("dummy-data")
    def dummy_data():
        from init_app import add_sl_domains, add_proton_partner

        LOG.w("reset db, add fake data")
        fake_data()
        add_sl_domains()
        add_proton_partner()
```

### 1.4 Default configuration keys (all from `example.env`, which the run used verbatim)

| Key | Value | `file:line` |
|-----|-------|-------------|
| `URL` | `http://localhost:7777` | `example.env:6` |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:22` |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:75` |
| `FLASK_SECRET` | `secret` | `example.env:77` |
| `EVENT_WEBHOOK` | **absent** → defaults to `None` | `app/config.py:612` — `EVENT_WEBHOOK = os.environ.get("EVENT_WEBHOOK", None)` |
| `MEM_STORE_URI` | **absent** → defaults to `None` | `app/config.py:568` — `MEM_STORE_URI = os.environ.get("MEM_STORE_URI", None)` |
| `DISABLE_RATE_LIMIT` | **absent** → defaults to `False` | `app/config.py:602` — `DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ` |

**OBSERVED:** `.env` is byte-identical to `example.env`; `EVENT_WEBHOOK`, `MEM_STORE_URI`, and `DISABLE_RATE_LIMIT` do not appear in `example.env` at all, so all three take their unset defaults. These three defaults determine the three headline findings: the event dispatch is a logged no-op (`EVENT_WEBHOOK=None`), the per-user Redis token bucket / Redlock guard are no-ops (`MEM_STORE_URI=None`), and route-level rate limiting is active (`DISABLE_RATE_LIMIT=False`).

### 1.5 Seeded and used accounts (OBSERVED)

- **id=1 `john@wick.com` / `password`** — `is_admin`, **PREMIUM** via an active monthly Subscription → **bypasses the free quota** (`User.can_create_new_alias` returns `True` for a `lifetime_or_active_subscription()` user, `app/models.py:878-879`) and uses the **PAID** rate-limit buckets. This is the primary happy-path user.
- **id=2 `winston@continental.com`** — ManualSubscription → effectively premium (not usable for the quota test).
- **id=3 `freebie@example.com` / `password`** — created at runtime as a genuinely **FREE** fixture (`max_alias_for_free_account=5`) specifically for the quota test (§7 B1).
- A `john` API key created for the API-contrast capture (§8): `pozipbnvelphbezslupbfqavqfgwoghijexzknglkplqvbvxedmlmrldopes`.

### 1.6 Observation taps (external, NO source instrumentation)

The investigation is fully non-invasive — **no source file was modified** to capture evidence:

- **PostgreSQL statement log:** `log_statement='all'`, `log_min_duration_statement=0` → `/var/log/postgresql/pg.log`. This captures every `BEGIN`/`SELECT`/`INSERT`/`UPDATE`/`COMMIT` emitted by the SQLAlchemy engine (`app/db.py`).
- **Dev-server stdout** → `/tmp/server.log` (the structured `LOG` object, `app/log.py`).
- **A `requests`-based HTTP client** drove the live web entry point (login → form submission), and `curl` drove the API contrast (§8).

Note: the in-memory flask-limiter route budget (5 create-POST/min, ~10 GET/min on `/dashboard/`) is ephemeral, so the dev server was restarted between capture batches to reset the counters.

---

## Section 2 — Q1: Run-and-Observe Walkthrough (OBSERVED, default config)

This is the "watch it live" narrative: log in as the seeded user, then create the first alias.

### 2.1 Login (OBSERVED)

The dashboard is behind Flask-Login (`@login_required` on `app/dashboard/views/index.py:56`), so a session must be established first. The captured client exchange:

```
GET /auth/login  -> extract hidden csrf_token
POST /auth/login {csrf_token, email=john@wick.com, password=password}
  -> HTTP 302, Location=http://localhost:7777/dashboard/
GET /dashboard/  -> HTTP 200  (page contains an /auth/logout link => authenticated)
```

The `POST /auth/login` returns **HTTP 302** redirecting to `http://localhost:7777/dashboard/`; the subsequent `GET /dashboard/` returns **HTTP 200** and the rendered page contains an `/auth/logout` link, confirming the session is authenticated.

### 2.2 Creating the first alias (OBSERVED)

With the session established, submitting the dashboard's **"Random Alias"** button (a plain HTML `POST` form, `templates/dashboard/index.html:50-56`) creates an alias and lands the browser back on the dashboard with a **green toastr success message** (`toastr.success("Alias … has been created")`). Behind that single click:

- the browser issues `POST /dashboard/` with `form-name=create-random-email` and a `csrf_token` (full request detail in **Section 3**);
- the `dashboard.index` view (`app/dashboard/views/index.py:55`) validates CSRF and quota, calls `Alias.create_new_random`, and commits (**Sections 4–5**);
- three tables are written in one transaction (**Section 5**);
- the application logs exactly four lines, including the gated event no-op (**Section 6**);
- the view responds **HTTP 302** (Post/Redirect/Get) to `…/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=` (**Section 4**);
- the browser follows the redirect with a `GET /dashboard/` (HTTP 200) that renders the flashed success message.

Error and edge conditions (quota, CSRF, rate limit, custom-alias validation, DB integrity) are catalogued in **Section 7**.

---

## Section 3 — Q2: The Frontend Request (OBSERVED, default config)

Both alias-creation controls are **plain HTML `POST` forms**, so the browser sends the default form encoding **`application/x-www-form-urlencoded`** (there is no JavaScript/JSON/XHR involved in the submission).

### 3.1 Form fields

**Random-alias forms** — rendered by `templates/dashboard/index.html`:

- hidden `form-name=create-random-email` at `templates/dashboard/index.html:52` (primary button), and again at `:70` and `:80` (the two dropdown variants);
- `csrf_token` at `templates/dashboard/index.html:41` (emitted as `{{ csrf_form.csrf_token }}`);
- the dropdown variants add a hidden `generator_scheme` at `templates/dashboard/index.html:72` and `:82`, whose values come from `AliasGeneratorEnum` (`app/models.py:211-213`):

```python
class AliasGeneratorEnum(EnumE):
    word = 1  # aliases are generated based on random words
    uuid = 2  # aliases are generated based on uuid
```

The primary "Random Alias" button form (`:50-56`) carries **no** `generator_scheme`, so it falls back to the user's default generator (see the route chain below).

**Custom-alias form** — rendered by `templates/dashboard/custom_alias.html`: fields `prefix`, `signed-alias-suffix`, `mailboxes`, `note`. **Note:** the custom form has **NO `form-name`** field (unlike the random forms).

### 3.2 RANDOM alias — `POST /dashboard/` — route `app/dashboard/views/index.py:55` (`dashboard.index`)

Captured request/response (OBSERVED):

```
REQUEST: POST http://localhost:7777/dashboard/
Content-Type: application/x-www-form-urlencoded
Body: csrf_token=<token>&form-name=create-random-email&generator_scheme=1
RESPONSE: HTTP 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=12&query=&sort=&filter=
Content-Length: 339 ; Set-Cookie: slapp=<session>; HttpOnly; Path=/; SameSite=Lax
```

Observed variants (the `generator_scheme` body field selects the address style):

- `generator_scheme=1` (word) → `<Alias 12 demure_pierce542@sl.local>`
- `generator_scheme=2` (uuid) → `<Alias 13 7a467a3a-3e32-438c-ad6b-1681e07d5306@sl.local>`
- absent (falls back to the user default) → `<Alias 14 strife_grated152@sl.local>`

The domain `sl.local` is `EMAIL_DOMAIN` (`example.env:22`).

**Route handling chain** (each step cited):

- `app/dashboard/views/index.py:57-61` — the route-level rate limiter, applied only to the create-random POST:
  ```python
  @limiter.limit(
      ALIAS_LIMIT,
      methods=["POST"],
      exempt_when=lambda: request.form.get("form-name") != "create-random-email",
  )
  ```
- `app/dashboard/views/index.py:85` — `csrf_form = CSRFValidationForm()` is built, and `:88-90` rejects an invalid CSRF before any write.
- `app/dashboard/views/index.py:97-104` — the `create-random-email` branch checks `current_user.can_create_new_alias()`, computes the scheme, and calls the creation core:
  ```python
  elif request.form.get("form-name") == "create-random-email":
      if current_user.can_create_new_alias():
          scheme = int(
              request.form.get("generator_scheme") or current_user.alias_generator
          )
          if not scheme or not AliasGeneratorEnum.has_value(scheme):
              scheme = current_user.alias_generator
          alias = Alias.create_new_random(user=current_user, scheme=scheme)
  ```
  This confirms the variant behavior above: `scheme = int(generator_scheme or current_user.alias_generator)`, and an out-of-range value falls back to `current_user.alias_generator`.
- `app/dashboard/views/index.py:108` — `Session.commit()` (the single commit for the whole write set).
- `app/dashboard/views/index.py:113-121` — `redirect(url_for("dashboard.index", highlight_alias_id=alias.id, query=query, sort=sort, filter=alias_filter))` → the HTTP 302 in the response block above.

### 3.3 CUSTOM alias — `POST /dashboard/custom_alias` — route `app/dashboard/views/custom_alias.py:30`

Captured GET (to read the form options) and the creation request/response (OBSERVED):

```
GET /dashboard/custom_alias -> 200
  signed-alias-suffix <select> options: "@old.com (your domain)",
      ".twelve516@premium.com (Premium domain)", ".flouts254@sl.local (Public domain)"
  mailboxes <select>: 1=john@wick.com, 2=pgp@example.org
REQUEST: POST http://localhost:7777/dashboard/custom_alias
Content-Type: application/x-www-form-urlencoded
Body: csrf_token=<token>&prefix=obs-custom-01&signed-alias-suffix=%40old.com.alUCEw.Pi_xtlDipdcqK34uIF5Ki5Lp81U&mailboxes=1&note=observation+note
RESPONSE: HTTP 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=16   (NO query/sort/filter, unlike random)
Content-Length: 273 ; Set-Cookie: slapp=<session>
```

Result: `<Alias 16 obs-custom-01@old.com>`.

**The `signed-alias-suffix` format** is `"{suffix}.{ts_b64}.{hmac}"` — an `itsdangerous` `TimestampSigner` signature. The signer is created at `app/alias_suffix.py:11`:

```python
signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
```

The three dot-separated parts in the observed value `%40old.com.alUCEw.Pi_xtlDipdcqK34uIF5Ki5Lp81U` (URL-decoded: `@old.com.alUCEw.Pi_xtlDipdcqK34uIF5Ki5Lp81U`) are the suffix (`@old.com`), the base64 timestamp (`alUCEw`), and the HMAC (`Pi_xtlDipdcqK34uIF5Ki5Lp81U`). The signature is verified server-side with a 600-second `max_age` (`app/alias_suffix.py:37-42`, `check_suffix_signature`), which is why a stale form fails with an "expired" message (§7 B5).

### 3.4 CSRF handling (Q2 sub-question)

Both web forms embed a `csrf_token` (Flask-WTF). The backend validates it via `CSRFValidationForm` **before any write** (`app/utils.py:157-158`):

```python
class CSRFValidationForm(FlaskForm):
    pass
```

- Random path: `app/dashboard/views/index.py:85` builds `CSRFValidationForm()`, and `:88-90` does `if not csrf_form.validate(): flash("Invalid request", "warning"); return redirect(request.url)`.
- Custom path: `app/dashboard/views/custom_alias.py:52` builds `CSRFValidationForm()`, and `:56-58` does the same `flash("Invalid request", "warning")` + redirect.

An empty `CSRFValidationForm` subclass is sufficient because Flask-WTF injects and validates the `csrf_token` field automatically. The invalid-CSRF runtime behavior is captured in §7 B2.

---

## Section 4 — Q3: The Backend Response (OBSERVED, default config)

### 4.1 Post/Redirect/Get is confirmed

Every web-form mutation returns **HTTP 302** with a flash message, and the browser then follows the redirect with a `GET` that returns **HTTP 200**. This is the classic Post/Redirect/Get (PRG) pattern and is confirmed for both the random and custom paths.

- **Random success** `Location` includes the highlight id plus empty query/sort/filter params: `http://localhost:7777/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=` — produced by `redirect(url_for("dashboard.index", highlight_alias_id=alias.id, query=query, sort=sort, filter=alias_filter))` at `app/dashboard/views/index.py:113-121`.
- **Custom success** `Location` is just `http://localhost:7777/dashboard/?highlight_alias_id=<id>` (**no** query/sort/filter) — produced by `redirect(url_for("dashboard.index", highlight_alias_id=alias.id))` at `app/dashboard/views/custom_alias.py:158-161`:
  ```python
                  Session.commit()
                  flash(f"Alias {full_alias} has been created", "success")

                  return redirect(url_for("dashboard.index", highlight_alias_id=alias.id))
  ```

### 4.2 Response metadata (OBSERVED)

| Metadata | Random `POST /dashboard/` | Custom `POST /dashboard/custom_alias` |
|----------|---------------------------|----------------------------------------|
| Status | `HTTP 302 FOUND` | `HTTP 302 FOUND` |
| `Location` | `…/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=` | `…/dashboard/?highlight_alias_id=<id>` |
| `Set-Cookie` | `slapp=<session>; HttpOnly; Path=/; SameSite=Lax` | `slapp=<session>` |
| `Content-Length` | `339` | `273` |

### 4.3 Flash-message mechanism

The flash is delivered via the session cookie and rendered at the top of the next page by `templates/base.html:98-103`:

```html
        {% with messages = get_flashed_messages(with_categories=true) %}
          <!-- Categories: success (green), info (blue), warning (yellow), danger (red) -->
          {% if messages %}

            {% for category, message in messages %}<script>toastr.{{category }}("{{ message }}");</script>{% endfor %}
          {% endif %}
        {% endwith %}
```

`get_flashed_messages(with_categories=true)` yields `(category, message)` pairs that are rendered as `<script>toastr.{{category}}("{{ message }}");</script>`. Categories map to colors: **success** (green), **info** (blue), **warning** (yellow), **danger** (red). Observed success flashes:

- Random: `toastr.success("Alias doting_ratios207@sl.local has been created")` — matches `flash(f"Alias {alias.email} has been created", "success")` at `app/dashboard/views/index.py:111`.
- Custom: `toastr.success("Alias obs-custom-01@old.com has been created")` — matches `flash(f"Alias {full_alias} has been created", "success")` at `app/dashboard/views/custom_alias.py:159`.

**Accuracy note:** a `toastr.success("Copied to clipboard")` call is also present in the page (`templates/base.html:170`), but it is a **static JS string** wired to the copy button — it is **NOT** a flash message and must not be misattributed to alias creation.

---

## Section 5 — Q4: Database Changes (OBSERVED via PostgreSQL `log_statement=all` + `psql` row diffs; default config)

### 5.1 The single creation core: `Alias.create`

Both entry points ultimately call `Alias.create` (`app/models.py:1627-1692`), which performs, in ONE call: the per-user rate check → email sanitize → global-trash lookups → `Session.add(new_alias)` (INSERT `alias`) → `DailyMetric.get_or_create_today_metric().nb_alias += 1` → **(gated)** `EventDispatcher.send_event(...)` (protobuf built at `app/models.py:1680`) → `emit_alias_audit_log(new_alias, AliasAuditLogAction.CreateAlias, "New alias created")` (INSERT `alias_audit_log`). The relevant tail of the method:

```python
        Session.add(new_alias)
        DailyMetric.get_or_create_today_metric().nb_alias += 1
        ...
        event = AliasCreated(
            id=new_alias.id,
            email=new_alias.email,
            note=new_alias.note,
            enabled=True,
            created_at=int(new_alias.created_at.timestamp),
        )
        EventDispatcher.send_event(user, EventContent(alias_created=event))
        emit_alias_audit_log(
            new_alias, AliasAuditLogAction.CreateAlias, "New alias created"
        )

        return new_alias
```

The view then issues a single `Session.commit()` (`app/dashboard/views/index.py:108`), so **all writes land in one transaction**.

### 5.2 RANDOM alias (id=17 `outdid_toiled140@sl.local`) — 3 tables

**Before/during/after row-count diff** (captured by `psql` counts around the operation):

| Table | Delta | Operation |
|-------|-------|-----------|
| `alias` | **+1** | INSERT |
| `alias_audit_log` | **+1** | INSERT |
| `daily_metric` | **+0** | **UPDATE** (today's row already existed) |
| `sync_event` | **0** | — (not touched) |
| `alias_mailbox` | **0** | — (not touched) |

The whole operation is ONE `BEGIN..COMMIT`: `BEGIN` → 10 `SELECT`s (user, subscription, alias-count [quota], contact, deleted_alias×2, subscription, deleted_alias, domain_deleted_alias, public_domain) → `INSERT alias` → `SELECT daily_metric` → `UPDATE daily_metric` → `INSERT alias_audit_log` → `COMMIT`. The exact, untruncated writes (OBSERVED):

```
INSERT INTO alias (created_at, updated_at, user_id, email, name, enabled, flags, custom_domain_id, automatic_creation, directory_id, note, mailbox_id, disable_pgp, cannot_be_disabled, disable_email_spoofing_check, batch_import_id, original_owner_id, pinned, transfer_token, transfer_token_expiration, hibp_last_check, last_email_log_id) VALUES ('2026-07-13T15:22:03.799304'::timestamp, NULL, 1, 'outdid_toiled140@sl.local', NULL, true, 0, NULL, false, NULL, NULL, 1, false, false, false, NULL, NULL, false, NULL, '2026-07-13T15:22:03.799331'::timestamp, NULL, NULL) RETURNING alias.id
UPDATE daily_metric SET updated_at='2026-07-13T15:22:03.804106'::timestamp, nb_alias=17 WHERE daily_metric.id = 1
INSERT INTO alias_audit_log (created_at, updated_at, user_id, alias_id, alias_email, action, message) VALUES ('2026-07-13T15:22:03.805305'::timestamp, NULL, 1, 17, 'outdid_toiled140@sl.local', 'create', 'New alias created') RETURNING alias_audit_log.id
```

### 5.3 CUSTOM alias with 2 mailboxes (id=18 `obs-two-mbx.lilies317@sl.local`) — 4 tables

**Before/during/after row-count diff:**

| Table | Delta | Operation |
|-------|-------|-----------|
| `alias` | **+1** | INSERT |
| `alias_audit_log` | **+1** | INSERT |
| `daily_metric` | **+0** | **UPDATE** (`nb_alias=18`) |
| `alias_mailbox` | **+1** | INSERT (the secondary mailbox) |
| `sync_event` | **0** | — (not touched) |

The extra (secondary) mailbox becomes an `alias_mailbox` row; the **primary** mailbox stays on `alias.mailbox_id`. This is the loop at `app/dashboard/views/custom_alias.py:152-156`:

```python
                for i in range(1, len(mailboxes)):
                    AliasMailbox.create(
                        alias_id=alias.id,
                        mailbox_id=mailboxes[i].id,
                    )
```

The exact writes (OBSERVED flush order):

```
INSERT INTO alias (... note, mailbox_id ...) VALUES ('2026-07-13T15:23:36.877601'::timestamp, NULL, 1, 'obs-two-mbx.lilies317@sl.local', ..., 'two mailbox test', 1, ...) RETURNING alias.id
INSERT INTO alias_audit_log (...) VALUES ('2026-07-13T15:23:36.880596'::timestamp, NULL, 1, 18, 'obs-two-mbx.lilies317@sl.local', 'create', 'New alias created') RETURNING alias_audit_log.id
UPDATE daily_metric SET updated_at='2026-07-13T15:23:36.881126'::timestamp, nb_alias=18 WHERE daily_metric.id = 1
INSERT INTO alias_mailbox (created_at, updated_at, alias_id, mailbox_id) VALUES ('2026-07-13T15:23:36.882472'::timestamp, NULL, 18, 2) RETURNING alias_mailbox.id
```

**INFERRED:** the random-vs-custom statement **order** differs (random did `UPDATE daily_metric` before `INSERT alias_audit_log`; custom emitted the `alias_audit_log` INSERT before the `daily_metric` UPDATE). This is SQLAlchemy unit-of-work flush ordering — an implementation detail. The **set of tables written is the canonical, stable fact**; the intra-transaction statement ordering is not.

### 5.4 Key facts (cited)

- **`daily_metric` is a GLOBAL per-day counter, NOT per-user** (`app/models.py:3262-3287`; `get_or_create_today_metric` at `:3280`). Its schema has `date` UNIQUE and columns `id, created_at, updated_at, date, nb_new_web_non_proton_user, nb_alias` — there is **no `user_id`**:
  ```python
  class DailyMetric(Base, ModelMixin):
      __tablename__ = "daily_metric"
      date = sa.Column(sa.Date, nullable=False, unique=True)
      nb_new_web_non_proton_user = sa.Column(...)
      nb_alias = sa.Column(sa.Integer, nullable=False, server_default="0", default=0)

      @staticmethod
      def get_or_create_today_metric() -> DailyMetric:
          today = arrow.utcnow().date()
          daily_metric = DailyMetric.get_by(date=today)
          if not daily_metric:
              daily_metric = DailyMetric.create(
                  date=today, nb_new_web_non_proton_user=0, nb_alias=0
              )
          return daily_metric
  ```
  The **first** alias of a calendar day INSERTs the row; **every later one** UPDATEs `nb_alias += 1`. In the runs above the row already existed (id=1), so both creations were UPDATEs.
- **`alias_audit_log` is written SYNCHRONOUSLY in the same transaction** (`app/models.py:3810` defines the model), via `emit_alias_audit_log` (`app/alias_audit_log_utils.py:18-32`) with `action='create'`, `message='New alias created'`:
  ```python
  def emit_alias_audit_log(alias, action, message, user_id=None, commit=False):
      AliasAuditLog.create(
          user_id=user_id or alias.user_id,
          alias_id=alias.id,
          alias_email=alias.email,
          action=action.value,   # AliasAuditLogAction.CreateAlias = "create"
          message=message,
          commit=commit,
      )
  ```
- **Across the ENTIRE `pg.log`: 0 `INSERT INTO sync_event` and 0 `NOTIFY` statements** — the `sync_event` table stayed empty. This independently corroborates the Q5 finding (§6) that the domain event is a no-op in the default configuration.

### 5.5 Table-count summary

- **Default random alias = 3 tables:** `alias` (INSERT), `daily_metric` (UPDATE, or INSERT on the day's first alias), `alias_audit_log` (INSERT).
- **Custom alias with a secondary mailbox = 4 tables:** the above three **plus** `alias_mailbox` (INSERT, one row per secondary mailbox). A single-mailbox custom alias writes the same 3 tables as a random alias.

---

## Section 6 — Q5: Logs & Background Work (OBSERVED, default config)

### 6.1 Complete application log for ONE random `POST /dashboard/`

Worker PID 29814, ~25ms of application work — **exactly 4 lines** (OBSERVED, verbatim from `/tmp/server.log`):

```
2026-07-13 15:25:48,269 - SL - DEBUG - 29814 - "app/models.py:1459" - generate_random_alias_email() - generate email avatar_lieder923@sl.local
2026-07-13 15:25:48,285 - SL - INFO  - 29814 - "app/events/event_dispatcher.py:62" - send_event() - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 15:25:48,289 - SL - DEBUG - 29814 - "app/dashboard/views/index.py:110" - index() - create new random alias <Alias 19 avatar_lieder923@sl.local> for user <User 1 John Wick john@wick.com>
2026-07-13 15:25:48,294 - SL - DEBUG - 29814 - "server.py:284" - after_request() - 127.0.0.1 POST /dashboard/ ImmutableMultiDict([]) 302, takes 0.04426836967468262
```

### 6.2 Complete application log for ONE custom `POST /dashboard/custom_alias`

**Exactly 2 lines** — the custom view emits **no** dedicated create-confirmation `LOG` line (only the random view logs `"create new random alias …"` at `app/dashboard/views/index.py:110`):

```
2026-07-13 15:26:14,568 - SL - INFO  - 29814 - "app/events/event_dispatcher.py:62" - send_event() - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 15:26:14,573 - SL - DEBUG - 29814 - "server.py:284" - after_request() - 127.0.0.1 POST /dashboard/custom_alias ImmutableMultiDict([]) 302, takes 0.056450605392456055
```

### 6.3 Findings

- **The only "event" line is a GATED NO-OP** at `app/events/event_dispatcher.py:61-65`. The `send_event` static method short-circuits when no webhook is configured:
  ```python
          if not config.EVENT_WEBHOOK and skip_if_webhook_missing:
              LOG.i(
                  "Not sending events because webhook is not configured and allowed to be empty"
              )
              return
  ```
  The log line at `app/events/event_dispatcher.py:62` (`send_event()`) is exactly this branch. The `AliasCreated` protobuf is built in `Alias.create` (`app/models.py:1680`) and then **discarded** — nothing is dispatched because `EVENT_WEBHOOK` is unset (`app/config.py:612` — `EVENT_WEBHOOK = os.environ.get("EVENT_WEBHOOK", None)`).
- **Running processes (via `ps`): ONLY the dev web server** (parent + Werkzeug-reloader child). There is **NO** `event_listener.py`, **NO** `job_runner.py`, and **NO** `cron.py` running.
- **The `alias_audit_log` write is SYNCHRONOUS** within the request transaction (see §5.4) — it is **not** a background job.
- **Q5 answer (stated plainly):** in the default configuration, alias creation triggers **NO background tasks and NO follow-up events**. The complete extra work beyond the `alias` INSERT is: one `daily_metric` UPDATE, one `alias_audit_log` INSERT, and one logged (discarded) event — all inside the same synchronous request transaction.
- **NON-CANONICAL contrast (INFERRED):** with `EVENT_WEBHOOK` (or a partner) configured, `send_event` would fall through the gate, serialize the protobuf (`event.SerializeToString()`), and `PostgresDispatcher.send` would `INSERT sync_event` + `NOTIFY simplelogin_sync_events` for the external `event_listener.py` consumer. This path was **not run** (it is non-default) and is therefore inferred from the code, not observed.

---

## Section 7 — Q6: Error / Edge-Path Catalog (OBSERVED live unless labelled; default config; user `john` unless noted)

**Up front:** each failing branch performs **NO database write unless explicitly stated**. The success write-set (§5) never occurs on a rejected request.

### B1 — Quota exceeded

Free user `freebie@example.com` (id=3, `max=5`). The gate is `User.can_create_new_alias` (`app/models.py:867-884`), which for a non-subscription user returns `Alias.filter_by(user_id=self.id).count() < self.max_alias_for_free_account()`:

```python
        if self.lifetime_or_active_subscription():
            return True
        else:
            return (
                Alias.filter_by(user_id=self.id).count()
                < self.max_alias_for_free_account()
            )
```

- **RANDOM** (`app/dashboard/views/index.py:122-123`): at count=5 → **HTTP 302**, `Location=http://localhost:7777/dashboard/?query=&sort=&filter=&page=0` (**NO `highlight_alias_id` ⇒ no write**), alias count stays **5**, flash `toastr.warning("You need to upgrade your plan to create new alias.")`.
- **CUSTOM** (`app/dashboard/views/custom_alias.py:36-42`): **HTTP 302** → flash `toastr.warning("You have reached free plan limit, please upgrade to create new aliases")`; count stays **5**. A maxed-out free user's `GET /dashboard/custom_alias` renders **0** `signed-alias-suffix` options.

### B2 — Invalid / missing CSRF

Path `app/dashboard/views/index.py:85-90`; validator `CSRFValidationForm` (`app/utils.py:157-158`). A wrong **or** missing `csrf_token` → **HTTP 302**, `Location=http://localhost:7777/dashboard/`, alias count **19→19** (**no write**), ~6ms, flash `toastr.warning("Invalid request")`:

```python
    if request.method == "POST":
        if not csrf_form.validate():
            flash("Invalid request", "warning")
            return redirect(request.url)
```

### B3 — Route rate-limit 429

Flask-Limiter with `ALIAS_LIMIT = "100/day;50/hour;5/minute"` (`app/config.py:448` — `ALIAS_LIMIT = os.environ.get("ALIAS_LIMIT") or "100/day;50/hour;5/minute"`). This is **ACTIVE** in the default in-memory configuration because `DISABLE_RATE_LIMIT=False` (`app/config.py:602`), and it is keyed by user id.

- **WEB:** POST #1–5 → HTTP 302 (aliases created); POST #6 → **HTTP 429**, `Content-Type: text/html`, **6067 bytes**. The rendered page's visible text is `429  Whoa, slow down there, pardner!  Home Page` (`templates/error/429.html`, which extends `error.html`). The log line is emitted at `server.py:364` (WARNING): `"Client hit rate limit on path /dashboard/, user:<User 1 John Wick john@wick.com>"`.
- **Handler** `server.py:362-372` decides HTML vs. JSON by path prefix:
  ```python
      @app.errorhandler(429)
      def rate_limited(e):
          LOG.w(
              "Client hit rate limit on path %s, user:%s",
              request.path,
              get_current_user(),
          )
          if request.path.startswith("/api/"):
              return jsonify(error="Rate limit exceeded"), 429
          else:
              return render_template("error/429.html"), 429
  ```

### B4 — Per-user Redis token bucket + Redlock parallel limiter

Files `app/rate_limiter.py` and `app/parallel_limiter.py`.

- **CANONICAL DEFAULT (runtime-confirmed):** `MEM_STORE_URI=None`, so the initialization gate at `server.py:163-165` never runs, leaving `rate_limiter.lock_redis=None` **and** `parallel_limiter.lock_redis=None`. Consequently the per-user token bucket and the Redlock guard are **BOTH NO-OPS by default**. The bucket returns immediately at `app/rate_limiter.py:28-29`:
  ```python
      if not lock_redis:
          return
  ```
  and the Redlock guard is likewise skipped (`app/parallel_limiter.py:51-52`). Thresholds (for reference, not hit in default config): `ALIAS_CREATE_RATE_LIMIT_FREE = [(10, 900), (50, 3600)]` and `ALIAS_CREATE_RATE_LIMIT_PAID = [(50, 900), (200, 3600)]` (`app/config.py:554-558`).
- **NON-CANONICAL component demos** (real `app/rate_limiter.py` code, explicitly labelled non-default because they wire up a Redis store by hand):
  - **CASE A (NON-CANONICAL):** `lock_redis=RedisStorage("redis://localhost:6379")`, `max_hits=3` → calls #1–3 OK, calls #4–5 raise `TooManyRequests` (429). This is the `value > max_hits` branch at `app/rate_limiter.py:30-40`:
    ```python
        try:
            value = lock_redis.incr(bucket_lock_name, bucket_seconds)
            if value > max_hits:
                ...
                raise werkzeug.exceptions.TooManyRequests()
    ```
  - **CASE B (NON-CANONICAL):** `lock_redis=` a **dead** Redis on `:6399` → `ERROR app/rate_limiter.py:42 - Cannot connect to redis`; the function then **returns normally with NO exception** — i.e., it **fails open**. This is the except clause at `app/rate_limiter.py:41-42`:
    ```python
        except (redis.exceptions.RedisError, AttributeError):
            LOG.e("Cannot connect to redis")
    ```

### B5 — Custom-alias validations

All **HTTP 302 no-write EXCEPT** the duplicate/deleted cases, which re-render at **HTTP 200**. OBSERVED (verbatim):

```
bad prefix "bad!!prefix"      -> 302; error "Only lowercase letters, numbers, dashes (-), dots (.) and underscores (_) are currently supported for alias prefix. Cannot be more than 40 letters"
".." dots (testdots. + .word) -> 302; error "Your alias can't contain 2 consecutive dots (..)"   (custom_alias.py:L103-105)
tampered/garbage/EXPIRED suffix -> 302; warning "Alias creation time is expired, please retry"   (alias_suffix.py:L37-42 check_suffix_signature catches itsdangerous.BadSignature[+SignatureExpired subclass] -> None)
MISSING signed-alias-suffix    -> 302; error "Unknown error, refresh the page" + LOG.w custom_alias.py:96 "Alias suffix is tampered"  (None -> AttributeError, non-BadSignature -> except Exception)
missing mailbox                -> 302; error "At least one mailbox must be selected"
tampered mailbox id=99999      -> 302; warning "Something went wrong, please retry"
duplicate OWN (obs-dup-test@old.com) -> HTTP 200 (re-render, NOT redirect); error "You already have this alias obs-dup-test@old.com"
recreate DELETED               -> HTTP 200 (re-render); error "You have deleted this alias before. You can restore it on old.com 'Deleted Alias' page"
```

Line-level corroboration of the strings above (verified against source):

- bad prefix → `flash(...)` at `app/dashboard/views/custom_alias.py:65` (guarded by `check_alias_prefix`).
- `".."` dots → `flash("Your alias can't contain 2 consecutive dots (..)", "error")` at `app/dashboard/views/custom_alias.py:104` (block `:103-105`).
- expired/tampered suffix → `if not suffix:` → `flash("Alias creation time is expired, please retry", "warning")` at `app/dashboard/views/custom_alias.py:93`; `check_suffix_signature` returns `None` on `itsdangerous.BadSignature` (`app/alias_suffix.py:41-42`), and `SignatureExpired` is a `BadSignature` subclass, so an expired signature is caught the same way.
- missing suffix → the `except Exception:` branch → `LOG.w("Alias suffix is tampered, user %s", ...)` at `:96` + `flash("Unknown error, refresh the page", "error")` at `:97`.
- missing mailbox → `flash("At least one mailbox must be selected", "error")` at `app/dashboard/views/custom_alias.py:86`.
- tampered mailbox → `flash("Something went wrong, please retry", "warning")` at `app/dashboard/views/custom_alias.py:81` (the mailbox-verification loop rejects a mailbox that does not exist / is not owned / is unverified).
- duplicate OWN → `flash(f"You already have this alias {full_alias}", "error")` at `app/dashboard/views/custom_alias.py:120` (application-level check via `Alias.get_by(email=full_alias)`).
- recreate DELETED → `flash(f"You have deleted this alias before. You can restore it on {custom_domain.domain} 'Deleted Alias' page", "error")` at `app/dashboard/views/custom_alias.py:128` (via `DomainDeletedAlias.get_by`).

**Delete-route note:** the delete/disable branch lives at `app/dashboard/views/index.py:125-155` (`form-name=delete-alias` or `disable-alias` + an `alias-id` field, rendered at `templates/dashboard/index.html:485`), and it is **exempt** from `ALIAS_LIMIT` (the limiter's `exempt_when` at `index.py:57-61` only arms for `form-name == "create-random-email"`). Deleting a custom-domain alias inserts 1 `domain_deleted_alias` row — which is what makes the "recreate DELETED" branch above reachable.

### B6 — IntegrityError → rollback

Component-level reproduction of the DB-level duplicate/race path, handled at `app/dashboard/views/custom_alias.py:146-150` (OBSERVED, verbatim):

```
first create flushed: id=37
IntegrityError RAISED on duplicate INSERT: duplicate key value violates unique constraint "gen_email_email_key"
Session.rollback() executed -> transaction aborted, no partial write
final alias count for this email: 0
```

The handler:

```python
                except IntegrityError:
                    LOG.w("Alias %s already exists", full_alias)
                    Session.rollback()
                    flash("Unknown error, please retry", "error")
                    return redirect(url_for("dashboard.custom_alias"))
```

This is the **DB-level** duplicate/race path (HTTP 302 + `"Unknown error, please retry"`), distinct from the **application-level** duplicate check in B5 (HTTP 200 re-render + `"You already have this alias …"`). The application check catches an existing alias *before* the INSERT; the `IntegrityError` path catches a race that slips past that check and is stopped by the `gen_email_email_key` unique constraint, with `Session.rollback()` guaranteeing no partial write.

### INFERRED sub-branches (code-referenced, NOT web-reproduced — labelled INFERRED)

- **`verify_prefix_suffix` returns `False`** → the `else` branch at `app/dashboard/views/custom_alias.py:162-164` flashes the exact string `flash("something went wrong", "warning")` (note: lowercase, no "please retry" — this is a *distinct* message from the B5 mailbox-tampered `"Something went wrong, please retry"` at `:81`). The guarding comment at `:162` is `# only happen if the request has been "hacked"`. **INFERRED** — this branch was not reproduced via the web because a well-formed signed suffix passes `verify_prefix_suffix`.
- **`validate_email` raising `EmailNotValidError`** → `flash(str(e), "error")` at `app/dashboard/views/custom_alias.py:107-113`. **INFERRED** — hard to reach in practice because the prefix character-rules (`check_alias_prefix`) reject bad input first.
- **SL-public-domain `DeletedAlias` recreation** → symmetric to the observed custom-domain path; the global-trash branch flashes the generic `general_error_msg` (`f"{full_alias} cannot be used"`) at `app/dashboard/views/custom_alias.py:135`. **INFERRED.**

---

## Section 8 — Web vs. API Contrast (OBSERVED)

Both entry points call the **same `Alias.create` core** (`app/models.py:1627-1692`), so the **database write-set is identical** (§5). Only the HTTP envelope differs.

| Aspect | WEB `POST /dashboard/` (`app/dashboard/views/index.py:55`) | API `POST /api/alias/random/new` (`app/api/views/new_random_alias.py:21`) |
|--------|-------------------------------------------------------------|----------------------------------------------------------------------------|
| Auth | session cookie `slapp` + CSRF token | header `Authentication: <key>` (`app/api/base.py:17`), **NO CSRF** |
| Success status | **HTTP 302** (Post/Redirect/Get) | **HTTP 201 CREATED** |
| Success body | redirect + `Location …highlight_alias_id=<id>` + toastr flash | JSON alias object |
| Rate-limit response | **HTTP 429 HTML** (`templates/error/429.html`) | **HTTP 429 JSON** `{"error": "Rate limit exceeded"}` |
| Quota response | **HTTP 302** + warning flash | **HTTP 400 JSON** |

### 8.1 API happy path (OBSERVED, verbatim)

```
$ curl -s -i -X POST http://localhost:7777/api/alias/random/new \
    -H "Authentication: pozipbnvelphbezslupbfqavqfgwoghijexzknglkplqvbvxedmlmrldopes" \
    -H "Content-Type: application/json" -d '{"note":"api-obs"}'
HTTP/1.0 201 CREATED
Content-Type: application/json
Server: Werkzeug/1.0.1 Python/3.10.20
{"alias":"vicars_crumby504@sl.local","email":"vicars_crumby504@sl.local","id":31,"enabled":true,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[...],"note":null,"pinned":false,...}
```

The success serializer at `app/api/views/new_random_alias.py:106-116` returns `jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))), 201`.

### 8.2 API rate-limit and quota

- **API rate-limit** (after 5/min) → **HTTP 429** `{"error": "Rate limit exceeded"}` — the `/api/`-prefixed branch of the 429 handler (`server.py:369-370`).
- **API quota exceeded** → **HTTP 400 JSON** (`app/api/views/new_random_alias.py:35-43`; the `return …, 400` is at `:42`). The exact body is code-visible:
  ```
  {"error": "You have reached the limitation of a free account with the maximum of <MAX_NB_EMAIL_FREE_PLAN> aliases, please upgrade your plan to create more aliases"}
  ```
  with `MAX_NB_EMAIL_FREE_PLAN` defaulting to 5. **INFERRED** — this exact JSON body is derived from reading `app/api/views/new_random_alias.py:38-41`; it was not re-captured at runtime in the default configuration.

---

## Section 9 — Observed vs. Inferred, Canonical vs. Non-canonical (Summary)

### OBSERVED (default / canonical) — captured at runtime

- **Q1:** login → dashboard (302 → 200), then a single click creates an alias and lands back on the dashboard with a green toastr success flash.
- **Q2:** the frontend sends `POST` with `Content-Type: application/x-www-form-urlencoded`; random body `csrf_token`, `form-name=create-random-email`, optional `generator_scheme`; custom body `csrf_token`, `prefix`, `signed-alias-suffix`, `mailboxes`, `note` (no `form-name`).
- **Q3:** **HTTP 302** Post/Redirect/Get + toastr flash; `Location …highlight_alias_id=<id>` (random adds `&query=&sort=&filter=`); `Set-Cookie: slapp=…; HttpOnly; Path=/; SameSite=Lax`; `Content-Length` 339 (random) / 273 (custom).
- **Q4:** write-set = **3 tables** for random (`alias` INSERT, `daily_metric` UPDATE, `alias_audit_log` INSERT) and **4 tables** for a custom alias with a secondary mailbox (adds `alias_mailbox` INSERT); `daily_metric` is a **global per-day** counter; `alias_audit_log` is written **synchronously**; **0** `sync_event` rows and **0** `NOTIFY`s across the whole log.
- **Q5:** the domain event is a **gated no-op** (one `INFO` log line); **no background workers** run (`event_listener.py`/`job_runner.py`/`cron.py` all absent from `ps`); the audit log is synchronous, not a job.
- **Q6:** the B1 (quota), B2 (CSRF), B3 (route 429 HTML), B5 (custom-alias validations), and B6 (IntegrityError rollback) outputs; and the **B4 canonical default** finding that the Redis token bucket + Redlock are no-ops (`MEM_STORE_URI=None`).
- **§8:** the API happy path returning **HTTP 201** JSON.

### INFERRED (code-derived, not observed)

- The intra-transaction **statement ordering** difference between the random and custom paths (SQLAlchemy flush ordering) — the *set* of tables is the canonical fact.
- The **webhook-configured** `sync_event` INSERT + `NOTIFY simplelogin_sync_events` path (`send_event` fall-through + `PostgresDispatcher.send`).
- The `verify_prefix_suffix`-`False` → `flash("something went wrong", "warning")` branch (`app/dashboard/views/custom_alias.py:162-164`).
- The `validate_email` → `EmailNotValidError` → `flash(str(e), "error")` branch (`app/dashboard/views/custom_alias.py:107-113`).
- The exact **API quota 400 JSON body** (`app/api/views/new_random_alias.py:38-41`).
- The SL-public-domain `DeletedAlias` recreation branch (symmetric to the observed custom-domain path).

### NON-CANONICAL (non-default, explicitly labelled)

- The **Redis-backed token-bucket demos** in §7 B4:
  - **CASE A** — a live `RedisStorage("redis://localhost:6379")` with `max_hits=3` producing `TooManyRequests` on calls #4–5.
  - **CASE B** — a dead Redis on `:6399` producing `ERROR app/rate_limiter.py:42 - Cannot connect to redis` and then **failing open** (returning normally with no exception).

  Both are non-canonical because the default configuration has `MEM_STORE_URI=None`, which makes the bucket a no-op (`app/rate_limiter.py:28-29`). They are included only to demonstrate the real behavior of the `check_bucket_limit` code when a store is present.

---

*End of investigation. The default-configuration behavior (3-table write, gated event no-op, 302 Post/Redirect/Get + flash) is the canonical answer; configuration-dependent behavior (a partner/webhook adds a `sync_event` INSERT + `NOTIFY`) and non-default component demonstrations are labelled INFERRED / NON-CANONICAL above.*
