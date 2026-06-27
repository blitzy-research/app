# SimpleLogin — Normal Alias-Creation Flow: A Code-Grounded Investigation

> **Repository:** SimpleLogin (`app`) · **Branch:** `app_2cd6ee777f8c` · **HEAD:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
>
> This document answers a single behavioral question — *what exactly happens when a user creates a new alias in SimpleLogin?* — covering the frontend request, the backend response, the database writes (and **how many tables are touched**), the follow-up/background work, and the failure modes.

---

## How to read this document (methodology)

This investigation follows two strict principles:

1. **Code is the source of truth.** Every behavioral claim below carries an inline `file:line` (or `file:Lstart-Lend`) citation that points at the actual repository source at HEAD `2cd6ee777f8c`. External material (the project's own `docs/api.md`, the public SimpleLogin API documentation) is used *only to corroborate* — it never overrides the code. Where a claim and a doc disagree, the code wins.
2. **Reasoning, not just conclusions.** Each requirement section (R1–R6) ends with a short **"Why / Reasoning"** note that explains *how* the cited code produces the stated behavior, so a reader can independently re-derive the answer.

**On running the application.** The canonical execution environment for this repository is the user-provided Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (Python 3.10 + the project's Poetry-locked dependencies + PostgreSQL + Redis). The documented standard development setup is reproduced in **R1**. Where a full live run is not reproducible in a given sandbox, the behavioral claims here are derived authoritatively from the source code — which is exactly what the governing rule requires. Critically, the schema's tables are defined by the ORM models in `app/models.py`, and the Alembic migrations applied by the documented setup are generated from those same models (see R1), so reading the table-write accounting straight from the model definitions and the view/`create()` bodies is authoritative for the running app.

**Scope of the "normal" case.** SimpleLogin exposes **four** alias-creation code paths (Flows A–D, enumerated in R2). Unless otherwise noted, the canonical *normal* case investigated throughout is **Flow A — the web dashboard custom-alias form, performed by the seeded development user `john@wick.com`, selecting a single mailbox, and not a Proton partner user.** Variants (extra mailboxes, API paths, partner users) are called out explicitly wherever they change the behavior.

---

## Table of contents

- [Overview of the system and the alias-creation flows](#overview-of-the-system-and-the-alias-creation-flows)
- [R1 — Running and observing the app (local-dev baseline)](#r1--running-and-observing-the-app-local-dev-baseline)
- [R2 — The frontend request (four flows)](#r2--the-frontend-request-four-flows)
- [R3 — The backend response](#r3--the-backend-response)
- [R4 — Database changes and the table count](#r4--database-changes-and-the-table-count)
- [R5 — Background work and follow-up events](#r5--background-work-and-follow-up-events)
- [R6 — Failure modes](#r6--failure-modes)
- [Consolidated failure-mode matrix](#consolidated-failure-mode-matrix)
- [Summary of key findings](#summary-of-key-findings)

---

## Overview of the system and the alias-creation flows

SimpleLogin is a **Python/Flask monolith managed with Poetry** (`pyproject.toml` declares `python = "^3.10"` and the project's dependency set). Its responsibilities are split cleanly by package:

- **Server-rendered dashboard views** live under `app/dashboard/` — e.g. the custom-alias form view at `app/dashboard/views/custom_alias.py:30` and the dashboard home at `app/dashboard/views/index.py:55`.
- **The versioned REST API** lives under `app/api/` — e.g. `app/api/views/new_custom_alias.py` and `app/api/views/new_random_alias.py`.
- **The ORM and all write logic** live in `app/models.py` — most importantly the `Alias` model and its `Alias.create()` classmethod at `app/models.py:1628`.
- **Asynchronous infrastructure** is split between the in-process event dispatcher at `app/events/event_dispatcher.py` and three **repository-root** worker entry points: `event_listener.py`, `job_runner.py`, and `cron.py`.

Four code paths can create an alias; they are documented in full in **R2**:

| Flow | Surface | Endpoint | Body |
|------|---------|----------|------|
| **A** | Web dashboard, custom alias | `POST /dashboard/custom_alias` | `application/x-www-form-urlencoded` |
| **B** | Web dashboard, one-click random | `POST /` (home) | `application/x-www-form-urlencoded` |
| **C** | REST API, custom alias | `POST /api/v3/alias/custom/new` (and v2) | `application/json` |
| **D** | REST API, random alias | `POST /api/alias/random/new` | optional JSON `note` + query params |

All four ultimately funnel into the same ORM write path, `Alias.create()` (`app/models.py:1628`): the dashboard custom flow (A) and the API custom flow (C) call it directly, while the random flows (B and D) call `Alias.create_new_random()`, which in turn calls `Alias.create()` directly (`app/models.py:1750`). (The related helper `Alias.create_new()` also funnels into `Alias.create()` at `app/models.py:1709`.) This shared write path is why the database-write accounting in **R4** applies across all four flows. The headline result, proven there, is that a normal single-mailbox dashboard creation writes **exactly three tables**.

---

## R1 — Running and observing the app (local-dev baseline)

### The documented standard setup

The project's `CONTRIBUTING.md` prescribes the standard development workflow. The relevant, verified steps are:

- **Prerequisite — Python 3.10.** `CONTRIBUTING.md:23` lists "Python 3.10 and poetry to manage dependencies". The project explicitly notes it has not made newer interpreters work: `CONTRIBUTING.md:236` reads "we haven't managed to make python 3.12 work". (This is why a sandbox running Python 3.12/3.13 cannot run the app and must rely on the source as truth.)
- **Copy the environment template:** `cp example.env .env` (`CONTRIBUTING.md:88`).
- **Point the app at a local PostgreSQL** by editing `DB_URI` (`CONTRIBUTING.md:91-94`); the sample value given there is `DB_URI=postgresql://myuser:mypassword@localhost:35432/simplelogin` (`CONTRIBUTING.md:94`). The shipped default in the template is `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin` (`example.env:75`).
- **Create the schema, seed data, and start the server:**

  ```bash
  alembic upgrade head && flask dummy-data && python3 server.py
  ```

  This single command line is given verbatim at `CONTRIBUTING.md:106`.
- **Open the dashboard and log in.** `CONTRIBUTING.md:109` states you can then open `http://localhost:7777` and "login with `john@wick.com / password` account."

### Where the seed user comes from

The `flask dummy-data` CLI command is defined in `server.py:490-495`; it calls `fake_data()` (`server.py:495`). `fake_data()` itself is defined at `app/fake_data.py:40` and creates the development user with `email="john@wick.com"` (`app/fake_data.py:45`) and `password="password"` (`app/fake_data.py:47`). This is the account used for every "normal case" observation in this document.

### The local-dev schema nuance (important for R4)

The configuration template does **not** define `EVENT_WEBHOOK` at all — it is **absent from `example.env`** (it has no entry in that file), while `FLASK_SECRET=secret` is present at `example.env:77`. This absence matters for R5.

More importantly for the table-count accounting in R4, it is worth separating the **documented standard command** from a **contributor note**, because they are not the same thing:

- The standard setup command at `CONTRIBUTING.md:106` brings the schema up with **`alembic upgrade head`** and then seeds it with `flask dummy-data` (which runs `fake_data()`, `app/fake_data.py:40`). So under the documented workflow the schema is created by the Alembic migrations under `migrations/`.
- Separately, the "Database migration" section adds a contributor note at `CONTRIBUTING.md:129` — *"It is created via `db.create_all()` (cf `fake_data()` method)"* — given as the reason the *local* database is not used to auto-generate new migration scripts.

That note should **not** be read as a description of the current `fake_data()` code. `fake_data()` (`app/fake_data.py:40`) seeds the development user and sample data but does **not** itself call `db.create_all()`, and a repository-wide search finds the only `db.create_all()` reference in a **commented-out** helper at `shell.py:19` (the surrounding `create_db()` is guarded by `if False:` and actually uses `flask_migrate.upgrade()`). What matters for R4 is therefore narrower and verifiable: the relevant tables are **defined by the ORM models in `app/models.py`**, and the Alembic migrations applied by the documented `alembic upgrade head` step (as well as by deploy/test) are generated from those same models — so the table *set* does not diverge between models and migrations, and reading the write accounting from the model definitions and the view/`create()` bodies is authoritative for the running schema.

### Why / Reasoning

Because the relevant tables are defined by the ORM models in `app/models.py` and the Alembic migrations applied by `alembic upgrade head` (`CONTRIBUTING.md:106`) are generated from those same models, the database-write accounting in **R4** — read straight from the model definitions and the view/`create()` bodies — is authoritative for the live application; the table set does not diverge between the models and the migrations. (The `db.create_all()` phrasing at `CONTRIBUTING.md:129` is a contributor note about why the local DB is not used to *generate* migrations, not a description of the current `fake_data()` code, which does not call `db.create_all()` — see above.) And because the project pins Python 3.10 and declares 3.12+ unsupported (`CONTRIBUTING.md:23`, `CONTRIBUTING.md:236`), the canonical way to *observe* the behavior is the supplied Docker image; in any environment where that live run is not reproducible, the verified source citations throughout this document are the authoritative substitute, consistent with the "code is truth" rule.


---

## R2 — The frontend request (four flows)

Four distinct entry points create an alias. Each is documented below with its endpoint, HTTP method, content-type, payload, and the decorators/guards that wrap it.

### Flow A — Web dashboard, custom alias (the primary "normal" case)

- **Request:** `POST /dashboard/custom_alias`, content-type **`application/x-www-form-urlencoded`** (a server-rendered HTML form submission).
- **Route:** `app/dashboard/views/custom_alias.py:30` — `@dashboard_bp.route("/custom_alias", methods=["GET", "POST"])`.
- **Decorators, in order** (`app/dashboard/views/custom_alias.py:31-33`):
  1. `@limiter.limit(ALIAS_LIMIT, methods=["POST"])` (`:31`) — HTTP-window rate limit (see R6).
  2. `@login_required` (`:32`) — the user must be authenticated via the session cookie.
  3. `@parallel_limiter.lock(name="alias_creation")` (`:33`) — a per-user concurrency lock (see R6).
- **Form fields parsed** (in the `POST` branch):
  - `prefix` — `request.form.get("prefix").strip().lower().replace(" ", "")` (`app/dashboard/views/custom_alias.py:59`); the user-chosen local part, normalized to lowercase with spaces removed.
  - `signed-alias-suffix` — `request.form.get("signed-alias-suffix")` (`:60`); a server-signed token encoding the chosen domain/suffix (see "Suffix options" below).
  - `mailboxes` — `request.form.getlist("mailboxes")` (`:61`); a **repeated** field, so multiple mailbox ids can be submitted.
  - `note` — `request.form.get("note")` (`:62`).
  - The **CSRF token**, validated by `csrf_form.validate()` at `app/dashboard/views/custom_alias.py:56` before any field is read.

### Flow B — Web dashboard, one-click random alias

- **Request:** `POST /` (the dashboard home page) carrying the form field `form-name=create-random-email` (plus an optional `generator_scheme`).
- **Route:** `app/dashboard/views/index.py:55` — `@dashboard_bp.route("/", methods=["GET", "POST"])` (the view function `index()` begins at `app/dashboard/views/index.py:67`).
- **Branch:** the random-alias action is selected by `request.form.get("form-name") == "create-random-email"` (`app/dashboard/views/index.py:97`). The chosen generator scheme is read from `generator_scheme` (falling back to the user's default) at `:99-103`, then the alias is created by `Alias.create_new_random(user=current_user, scheme=scheme)` (`app/dashboard/views/index.py:104`).

### Flow C — REST API, custom alias

There are **two** custom-alias API endpoints — `v2` and `v3` — and their request bodies are **not** the same. Both use body **`application/json`**, are authenticated via the **`Authentication` header** carrying the API code, and accept an optional `?hostname` query parameter that records where the alias is used.

- **Endpoints, auth, and routes:** `POST /api/v2/alias/custom/new` (`new_custom_alias_v2`) and `POST /api/v3/alias/custom/new` (`new_custom_alias_v3`) — routes at `app/api/views/new_custom_alias.py:28` (v2) and `:115` (v3), each guarded by `@require_api_auth` (`:30` / `:117`). The `?hostname` query parameter is read at `:58` (v2) and `:147` (v3).
- **v2 body — `alias_prefix`, `signed_suffix`, and optional `note` only.** The v2 handler parses exactly these from the JSON body (`request.get_json()` at `:60`; `alias_prefix` `:64`, `signed_suffix` `:65`, `note` `:66`; see `app/api/views/new_custom_alias.py:60-67`). It accepts **no** mailbox list and **no** name — the alias is created against the account's **default** mailbox: `Alias.create(... mailbox_id=user.default_mailbox_id, ...)` (`app/api/views/new_custom_alias.py:96-100`).
- **v3 body — additionally `mailbox_ids` (an array) and optional `name`.** The v3 handler reads `mailbox_ids`, `note`, and `name` from the body (`request.get_json()` `:149`; dict-shape check `:153-154`; `mailbox_ids` `:160`, `note` `:161`, `name` `:162`; see `app/api/views/new_custom_alias.py:149-162`), validates the list (`:171-181`), creates the alias against the **first** selected mailbox — `Alias.create(... name=name or None, mailbox_id=mailboxes[0].id)` (`app/api/views/new_custom_alias.py:211-217`) — and then inserts one `alias_mailbox` row per extra mailbox in the loop at `:220-224`.
- **Corroboration (not authority):** the project's `docs/api.md` lists the v3 endpoint (`docs/api.md:23`), states the `Authentication`-header convention (`docs/api.md:73`), and explains the `hostname` query string (`docs/api.md:75-76`); the v3 custom-alias section documents the JSON body together with its `alias_prefix` / `signed_suffix` / `mailbox_ids` / optional `note` / optional `name` fields (`docs/api.md:381-394`). This independently matches the code, but the code at the cited lines remains the authoritative contract.

### Flow D — REST API, random alias

- **Request:** `POST /api/alias/random/new`, authenticated via the `Authentication` header. It accepts optional query parameters `?mode=word|uuid` and `?hostname`, **plus an optional JSON body carrying `note`**. The handler reads the body defensively — `note = None`, then `data = request.get_json(silent=True)`, then `if data: note = data.get("note")` (`app/api/views/new_random_alias.py:45-48`) — so a request body is not *required*, but a `note` is honored when one is supplied. The generator scheme is taken from the `?mode` query param (or the user default). This matches `docs/api.md:408-409`, which documents an optional `application/json` body with an optional `note`.
- **Route:** `app/api/views/new_random_alias.py:21`, guarded by `@require_api_auth` (`:23`). The `mode` parameter is parsed at `app/api/views/new_random_alias.py:97-104`; an invalid mode returns `400` (`:104`). `?hostname` is read at `:53`. The alias is created by `Alias.create_new_random(user=user, scheme=scheme, note=note)` (`app/api/views/new_random_alias.py:106`).

### Suffix options are fetched *before* creation

For the custom flows, the chosen domain/suffix is not free text — it is a **server-signed token**. The dashboard pre-loads the available suffixes via `get_alias_suffixes(current_user)` (`app/alias_suffix.py:94`, called from `app/dashboard/views/custom_alias.py:45`), and the API exposes them via `app/api/views/alias_options.py` (`GET /v4/alias/options` at `:13`, `GET /v5/alias/options` at `:77`). The selected suffix is signed; the create endpoints later verify the signature with `check_suffix_signature()` (`app/alias_suffix.py:37`).

### Why / Reasoning

The two surfaces differ because they target different clients. The **dashboard** is a browser-facing, server-rendered HTML form, so it uses `application/x-www-form-urlencoded` and is protected by a CSRF token (`app/dashboard/views/custom_alias.py:56`) and a session login (`:32`). The **API** is a machine-facing JSON contract authenticated by an `Authentication` header (`@require_api_auth`, `app/api/views/new_custom_alias.py:30`). In both cases the alias suffix arrives as a *signed* token rather than as a plain domain string: the server signs the suffix when it serves the options, and verifies that signature at creation time (`app/alias_suffix.py:37`), which ties the chosen domain/suffix to a specific user and a timestamp and prevents a client from fabricating a suffix for a domain it is not entitled to use.


---

## R3 — The backend response

### Dashboard (Flows A and B): `302` redirect + flash, not JSON

On success the dashboard does **not** return JSON. It returns an **HTTP `302` redirect** together with a flash message, following the Post/Redirect/Get (PRG) pattern:

- **Flow A:** `flash(f"Alias {full_alias} has been created", "success")` (`app/dashboard/views/custom_alias.py:159`), then `return redirect(url_for("dashboard.index", highlight_alias_id=alias.id))` (`app/dashboard/views/custom_alias.py:161`). The browser is redirected to the dashboard index with the new alias's id in the `highlight_alias_id` query parameter, and the index re-renders with that alias highlighted.
- **Flow B:** the analogous `flash(f"Alias {alias.email} has been created", "success")` (`app/dashboard/views/index.py:111`) and `redirect(...)` to `dashboard.index` with `highlight_alias_id=alias.id` (`app/dashboard/views/index.py:113-121`).

### API (Flows C and D): `201` with a JSON alias object

On success the API returns **HTTP `201`** with a JSON body. The body is assembled by the endpoint itself, which injects a top-level `alias` key and then spreads the serializer output:

- **Flow C (custom):** `jsonify(alias=full_alias, **serialize_alias_info_v2(get_alias_info_v2(alias)))` with status `201` — at `app/api/views/new_custom_alias.py:110-111` (v2) and `app/api/views/new_custom_alias.py:233-234` (v3).
- **Flow D (random):** `jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias)))` with status `201` — at `app/api/views/new_random_alias.py:115-116`.

The remaining keys are produced by `serialize_alias_info_v2()` (`app/api/serializer.py:55-92`). The full response shape is:

| Key | Source line | Notes |
|-----|-------------|-------|
| `alias` | endpoint (`app/api/views/new_custom_alias.py:110` / `:233`; `app/api/views/new_random_alias.py:115`) | **Injected by the endpoint**, not the serializer — the full alias email |
| `id` | `app/api/serializer.py:58` | Alias primary key |
| `email` | `app/api/serializer.py:59` | The alias address |
| `creation_date` | `app/api/serializer.py:60` | `created_at.format()` |
| `creation_timestamp` | `app/api/serializer.py:61` | `created_at.timestamp` |
| `enabled` | `app/api/serializer.py:62` | |
| `note` | `app/api/serializer.py:63` | |
| `name` | `app/api/serializer.py:64` | |
| `nb_forward` | `app/api/serializer.py:66` | Activity counter |
| `nb_block` | `app/api/serializer.py:67` | Activity counter |
| `nb_reply` | `app/api/serializer.py:68` | Activity counter |
| `mailbox` | `app/api/serializer.py:70` | `{id, email}` of the primary mailbox |
| `mailboxes` | `app/api/serializer.py:71-74` | array of `{id, email}` |
| `support_pgp` | `app/api/serializer.py:75` | |
| `disable_pgp` | `app/api/serializer.py:76` | |
| `latest_activity` | `app/api/serializer.py:77` | **`null` for a newly created alias** |
| `pinned` | `app/api/serializer.py:78` | |

`latest_activity` is initialized to `None` at `app/api/serializer.py:77` and is only replaced with a real object inside `if alias_info.latest_email_log:` (`app/api/serializer.py:80-92`). A brand-new alias has no `EmailLog` rows yet, so that branch is skipped and the key stays `null`.

### Why / Reasoning

The two response shapes follow from the two audiences. The dashboard targets a **browser**, so it uses the PRG pattern — a `302` to a GET URL (`app/dashboard/views/custom_alias.py:161`) plus a one-shot flash (`:159`) — which prevents duplicate submissions on refresh and lets the index page highlight the freshly created alias. The API targets **programmatic clients**, so it returns a machine-readable `201 Created` with a full JSON resource representation (`app/api/views/new_custom_alias.py:110-111`). The `latest_activity: null` value is not a special case in the response code — it falls out naturally because the serializer only populates activity when `latest_email_log` exists (`app/api/serializer.py:80`), and a just-created alias has none.


---

## R4 — Database changes and the table count

This is the headline question: *which records are inserted/updated, how many tables are touched, and are related entities created in the same operation?* The answer comes from walking `Alias.create()` in `app/models.py` and then the dashboard view that calls it.

### What `Alias.create()` does, in order

`Alias.create()` is defined at `app/models.py:1628`. Reading its body top to bottom:

1. **Rate-limit check (a read, not a write).** It picks the per-tier limit — `ALIAS_CREATE_RATE_LIMIT_PAID` for premium users, else `ALIAS_CREATE_RATE_LIMIT_FREE` (`app/models.py:1634-1637`) — and, for each configured bucket, calls `rate_limiter.check_bucket_limit(key, limit[0], limit[1])` (`app/models.py:1639-1641`). This consults Redis; it does not write a table (see R6).
2. **Trash lookups (reads).** `DeletedAlias.get_by(email=...)` (`app/models.py:1648`) and `DomainDeletedAlias.get_by(email=...)` (`app/models.py:1651`) — if the address was previously deleted, the method raises `AliasInTrashError`. Reads only.
3. **Custom-domain detection (read).** `Alias.get_custom_domain(email)` (`app/models.py:1656`) — read only.
4. **INSERT `alias`.** `Session.add(new_alias)` at **`app/models.py:1660`** — the alias row itself.
5. **INSERT/UPDATE `daily_metric`.** `DailyMetric.get_or_create_today_metric().nb_alias += 1` at **`app/models.py:1661`**. `get_or_create_today_metric()` (`app/models.py:3280`; table `daily_metric` at `app/models.py:3269`) inserts today's metric row if it does not yet exist and otherwise increments the existing one.
6. **Conditional UPDATE `users` (partner only).** Only when the alias is partner-created does it set the partner flag on the user (`app/models.py:1663-1667`). Not exercised in normal dev.
7. **Gated event dispatch.** `EventDispatcher.send_event(...)` at **`app/models.py:1687`** — this writes *nothing* in normal dev (the gating is detailed in R5).
8. **INSERT `alias_audit_log` (unconditional).** `emit_alias_audit_log(new_alias, AliasAuditLogAction.CreateAlias, "New alias created")` at **`app/models.py:1688-1690`**. That helper (`app/alias_audit_log_utils.py:18`) calls `AliasAuditLog.create(... action="create" ..., commit=commit)` (`app/alias_audit_log_utils.py:25-32`); the enum value `"create"` is defined at `app/alias_audit_log_utils.py:8`, and the table `alias_audit_log` at `app/models.py:3813`.

### How the dashboard view completes the unit of work

Crucially, `Alias.create()` is invoked by the dashboard with **neither `commit` nor `flush`** set. `Alias.create()` **overrides** the shared `ModelMixin.create` classmethod and independently pops both keyword arguments with a default of `False` — `commit = kw.pop("commit", False)` at `app/models.py:1629` and `flush = kw.pop("flush", False)` at `:1630` (this mirrors the general `ModelMixin.create` pattern at `app/models.py:116-119`, but the defaults that actually apply on this call path are the ones in the override). So the alias INSERT, the `daily_metric` change, and the `alias_audit_log` INSERT all accumulate in the SQLAlchemy session and are committed *together* by the view.

In `app/dashboard/views/custom_alias.py` the primary mailbox is stored directly on the alias via `mailbox_id=mailboxes[0].id` (`:143`); then `Session.flush()` (`:145`); then a loop `for i in range(1, len(mailboxes))` creates one `AliasMailbox` row per **extra** mailbox (`:152-156`; the `alias_mailbox` table/class at `app/models.py:2939`); and finally `Session.commit()` (`:158`) commits everything atomically.

Every row also inherits the columns from `ModelMixin` (`app/models.py:62`): an autoincrement `id` (`:63`), a non-null `created_at` defaulting to `arrow.utcnow` (`:64`), and an `updated_at` set `onupdate` (`:65`); the declarative base is `Base = declarative_base()` (`app/models.py:52`).

### ★ Headline answer

> **For a normal single-mailbox alias created by a non-partner development user (e.g. `john@wick.com`) via the dashboard, exactly THREE tables are written: `alias`, `alias_audit_log`, and `daily_metric`.**
>
> The **related entities created in the same transaction** are the `alias_audit_log` row (`app/models.py:1688-1690`) and the `daily_metric` row (`app/models.py:1661`; inserted once per day, updated on every subsequent creation that day). All three are committed together by the single `Session.commit()` at `app/dashboard/views/custom_alias.py:158`.

### Complete table of writes (always + conditional)

| Table | When written | Operation | Locator |
|-------|--------------|-----------|---------|
| `alias` | **Always** | INSERT | `app/models.py:1660` |
| `alias_audit_log` | **Always** | INSERT | `app/models.py:1688-1690`; `app/alias_audit_log_utils.py:18-32` |
| `daily_metric` | **Always** (today's row) | INSERT if absent, else UPDATE | `app/models.py:1661`, `:3280` |
| `alias_mailbox` | Only when ≥ 2 mailboxes selected | INSERT per extra mailbox | `app/dashboard/views/custom_alias.py:152-156` |
| `sync_event` | Only if a Proton partner user **and** `EVENT_WEBHOOK` configured | INSERT + `NOTIFY` | `app/events/event_dispatcher.py:25-26` |
| `users` | Only for partner-created aliases | UPDATE (flag) | `app/models.py:1663-1667` |
| `api_key` | Only on authenticated API paths | UPDATE `last_used`, `times` | `app/api/base.py:30-32` |
| `alias_used_on` | Only when an API call includes `?hostname` | INSERT | `app/api/views/new_custom_alias.py:106` / `:229`; `app/api/views/new_random_alias.py:111` |

### Flow A end-to-end

```mermaid
sequenceDiagram
    participant U as Browser (Dashboard)
    participant V as custom_alias view
    participant M as Alias.create()
    participant DB as PostgreSQL
    U->>V: POST /dashboard/custom_alias (prefix, signed-alias-suffix, mailboxes[], note, csrf)
    V->>V: Validate quota, CSRF, prefix, suffix signature, mailboxes, duplicates
    V->>M: Alias.create(user_id, email, note, mailbox_id)
    M->>DB: INSERT alias
    M->>DB: INSERT or UPDATE daily_metric (nb_alias += 1)
    M->>DB: INSERT alias_audit_log (action=create)
    M-->>V: new alias
    V->>DB: AliasMailbox.create() per EXTRA mailbox
    V->>DB: Session.commit()
    V-->>U: 302 redirect to /dashboard/?highlight_alias_id=ID + flash
```

### Why / Reasoning

The codebase uses a classic **unit-of-work** pattern: `Session.add` stages rows, `Session.flush` sends pending SQL without ending the transaction, and a single `Session.commit` makes everything durable (with `Session.rollback` on error — see R6). The **three always-written tables fall directly out of the three unconditional statements** in `Alias.create()` plus the view's commit: the alias INSERT (`app/models.py:1660`), the daily-metric increment (`:1661`), and the audit-log INSERT (`:1688-1690`). Each of the remaining rows is conditional for a concrete reason: `alias_mailbox` only exists to associate *additional* mailboxes beyond the primary one stored on `alias.mailbox_id` (so it is written only when ≥ 2 mailboxes are chosen, `app/dashboard/views/custom_alias.py:152-156`); `sync_event` is gated behind partner + webhook configuration (R5); the `users` flag update only applies to partner-created aliases (`app/models.py:1663-1667`); the `api_key` update is a side effect of API authentication, not of alias creation per se (`app/api/base.py:30-32`); and `alias_used_on` is only recorded when an API caller tells the server where the alias will be used via `?hostname`.


---

## R5 — Background work and follow-up events

The question here is whether alias creation triggers background tasks, emits follow-up events, or does work beyond the initial write. The answer has two parts: an **in-process event dispatch** (which is gated off in normal dev) and a set of **separate background processes** (which are unrelated to alias creation).

### The in-process event dispatcher is triple-gated

`Alias.create()` calls `EventDispatcher.send_event(...)` at `app/models.py:1687`. That method lives in `app/events/event_dispatcher.py`, where the notification channel name is `NOTIFICATION_CHANNEL = "simplelogin_sync_events"` (`app/events/event_dispatcher.py:14`). `EventDispatcher.send_event(...)` (`app/events/event_dispatcher.py:48-84`) returns early — **logging only, writing nothing** — if **any** of three gates is hit:

1. **Webhook explicitly disabled.** `if config.EVENT_WEBHOOK_DISABLE:` → `LOG.i("Not sending events because webhook is disabled")` then `return` (`app/events/event_dispatcher.py:57-59`).
2. **Webhook not configured.** `if not config.EVENT_WEBHOOK and skip_if_webhook_missing:` → `LOG.i("Not sending events because webhook is not configured ...")` then `return` (`app/events/event_dispatcher.py:61-65`). `EVENT_WEBHOOK` defaults to `None` (`app/config.py:612`, `os.environ.get("EVENT_WEBHOOK", None)`) and is **absent from `example.env`**, so this gate is active by default in dev.
3. **No Proton partner user.** If the user has no associated `PartnerUser`, `LOG.i("Not sending events because there's no partner user for user ...")` then `return` (`app/events/event_dispatcher.py:68-70`).

Only when **all** gates pass does the dispatcher actually persist anything: `PostgresDispatcher.send()` writes a `sync_event` row via `SyncEvent.create(content=event, flush=True)` (`app/events/event_dispatcher.py:25`) and issues `Session.execute("NOTIFY simplelogin_sync_events, '<id>';")` (`app/events/event_dispatcher.py:26`).

**Conclusion for normal dev:** the seeded user `john@wick.com` has no partner user, `EVENT_WEBHOOK` is unset, and `EVENT_WEBHOOK_DISABLE` is unset, so the `AliasCreated` event path is a **no-op that only logs** — there is **no `sync_event` row and no `NOTIFY`**. (This is also why the headline table count in R4 is three, not four.)

### Synchronous vs. asynchronous side effects

- **Synchronous (in the same transaction):** the `alias_audit_log` INSERT (`app/models.py:1688-1690`) and the `daily_metric` increment (`app/models.py:1661`). These are committed together with the alias by the view's `Session.commit()` (`app/dashboard/views/custom_alias.py:158`).
- **Potentially asynchronous (but absent in dev):** the gated `sync_event` row + `NOTIFY` (`app/events/event_dispatcher.py:25-26`). Even when enabled, the write is still part of the same DB transaction; only the downstream *consumption* of the notification is asynchronous.

### Separate background processes (unrelated to the alias INSERT)

Three long-running worker entry points exist at the **repository root** — not under `app/`:

- **`event_listener.py`** (root) — the PostgreSQL `LISTEN`/`NOTIFY` consumer of the `simplelogin_sync_events` channel. It reads `EVENT_LISTENER_DB_URI` (`event_listener.py:5`) and drives the root-level `events/` package (`from events.runner import Runner` at `event_listener.py:8`, `PostgresEventSource` at `event_listener.py:9`, instantiated with `PostgresEventSource(EVENT_LISTENER_DB_URI)` at `event_listener.py:35`). It only has work to do when `sync_event` rows are actually produced — which, per the gating above, does not happen for a normal dev alias creation.
- **`job_runner.py`** (root) — runs scheduled jobs; its module docstring is "Run scheduled jobs." (`job_runner.py:1-4`), and it handles things like the GDPR `ExportUserDataJob`. It is **not** part of, and is **not** synchronously triggered by, a normal dashboard alias creation — but it is **not wholly unrelated to aliases**: one of its job branches handles `JOB_SEND_ALIAS_CREATION_EVENTS` (`job_runner.py:295-302`), calling `send_alias_creation_events_for_user(...)` (`app/jobs/event_jobs.py:9`) to emit `AliasCreated` events for a user's aliases. That job is enqueued only by the Proton initial-alias-sync path (`_initial_alias_sync`, `app/proton/proton_callback_handler.py:46-52`) — never by the dashboard create flow.
- **`cron.py`** (root) — scheduled/periodic tasks. Unrelated to alias creation.

None of these three processes runs as part of — or is synchronously triggered by — a normal alias creation.

### Why / Reasoning

Event sourcing in SimpleLogin is deliberately **opt-in**: it only matters for Proton-partner integrations, so the dispatcher refuses to write a `sync_event` unless a webhook is configured *and* the user is a partner user (`app/events/event_dispatcher.py:57-70`). That is why the default development experience emits no events at all. By contrast, the audit-log and the daily metric are written **inside the same transaction** as the alias (`app/models.py:1661`, `:1688-1690`) precisely so they can never diverge from the alias row — if the alias commit rolls back, so do they. The root-level workers are independent services with their own database connections and lifecycles; they consume work that *other* code enqueues or notifies, and a normal dashboard alias creation enqueues none for them — there is no `sync_event`/`NOTIFY`, and `job_runner.py`'s `JOB_SEND_ALIAS_CREATION_EVENTS` job is enqueued only by the Proton initial-alias-sync path (`app/proton/proton_callback_handler.py:46-52`), never by the create flow.


---

## R6 — Failure modes

Failures fall into four categories. For each, the table below and the prose map the failure to its **API/HTTP surface**, its **log line**, and the **resulting database state**. Throughout, `LOG` is the `"SL"` logger (`app/log.py:79`) whose shortcuts `d`/`i`/`w`/`e` are aliased to `debug`/`info`/`warning`/`exception` at `app/log.py:74-77`.

### (a) Dashboard validation failures (Flow A) — error flash, no write

Each of these fails *before* any row is staged, so the database is untouched. **Most** branches re-flash an error and `return redirect(...)` back to the form, producing an HTTP `302`. The **duplicate / trash pre-insert checks are the exception**: they `flash` *without* a `return`, so control falls through to the `render_template(...)` at `app/dashboard/views/custom_alias.py:166`, producing an **inline re-render with HTTP `200`** (no redirect). Either way, no row is written:

- **Quota exceeded:** `if not current_user.can_create_new_alias():` → `LOG.d("%s can't create new alias", ...)`, a warning flash, and a redirect to the dashboard (`app/dashboard/views/custom_alias.py:36-42`).
- **Invalid CSRF:** `if not csrf_form.validate():` → "Invalid request" flash + redirect (`app/dashboard/views/custom_alias.py:56-58`).
- **Bad prefix** (not lowercase letters/numbers/`-`/`.`/`_`, or longer than 40 characters): `check_alias_prefix()` returns false → error flash + redirect (`app/dashboard/views/custom_alias.py:64`; the length/charset rules live in `app/alias_utils.py:418-425`).
- **Bad/unverified/foreign mailbox:** a mailbox that is missing, not owned by the user, or unverified → "Something went wrong" flash + redirect (`app/dashboard/views/custom_alias.py:74-87`).
- **Expired suffix:** `check_suffix_signature()` returns falsy → `LOG.w("Alias creation time expired for %s", ...)`, flash, redirect (`app/dashboard/views/custom_alias.py:90-94`).
- **Tampered suffix:** signature verification raises → `LOG.w("Alias suffix is tampered, user %s", ...)`, flash, redirect (`app/dashboard/views/custom_alias.py:95-98`).
- **Consecutive dots:** `".." in full_alias` → flash + redirect (`app/dashboard/views/custom_alias.py:104-105`).
- **Invalid email:** `validate_email(...)` raises `EmailNotValidError` → flash + redirect (`app/dashboard/views/custom_alias.py:108-113`).
- **Duplicate / trash collisions (pre-insert checks):** an existing `Alias` (`app/dashboard/views/custom_alias.py:117-122`), a `DomainDeletedAlias` (`:123-132`), or a `DeletedAlias` (`:134-135`) → error flash, **no insert**. Unlike every branch above, these three `flash` *without* a `return`, so control falls through to `render_template(...)` (`:166`) and the server responds with an **inline re-render (HTTP `200`)**, not a `302` redirect. (This is distinct from the `IntegrityError` race handler at `:146-150`, which *does* `return redirect(...)` → `302`; see (b) below.)

### (b) Database error at INSERT (race / duplicate) — `IntegrityError` → rollback, no partial row

If two requests race past the pre-insert duplicate check, the INSERT itself can violate a uniqueness constraint. The view wraps `Alias.create(...)` + `Session.flush()` in a `try/except IntegrityError` (`app/dashboard/views/custom_alias.py:146`): on failure it logs `LOG.w("Alias %s already exists", ...)` (`:147`), calls `Session.rollback()` (`:148`), flashes an error (`:149`), and redirects (`:150`). **Resulting DB state: no partial row** — the rollback discards the pending alias INSERT (and, because the daily-metric and audit-log writes were staged in the same uncommitted session, they are discarded too). This is exactly why the table count stays consistent: writes are all-or-nothing.

> **Dev-server note (concurrency).** The `Session.commit()` at `app/dashboard/views/custom_alias.py:158` sits *outside* the `try/except IntegrityError` (which wraps only `Alias.create()` + `Session.flush()` at `:138-145`). Under two *truly concurrent* same-email submissions, the threaded Flask **development** server shares a single scoped `Session` across request threads, so the losing request's `Session.rollback()` (`:148`) can invalidate the transaction the winning request then commits at `:158`, surfacing an unhandled `InvalidRequestError` as an HTTP `500`. This is strictly a development-server artifact — production runs Gunicorn with per-worker sessions and does not share-session-corrupt — and in **all** cases **no partial or duplicate row is persisted** (the rollback still guarantees the all-or-nothing outcome above).

### (c) API structured errors (Flow C) — explicit status codes

The API returns explicit JSON errors with the appropriate status. Representative locators from `app/api/views/new_custom_alias.py` (v2 / v3):

- **`201`** success — `:111` / `:234`.
- **`409`** duplicate (`"alias ... already exists"`) — `:88` / `:203`.
- **`412`** **expired _or_ tampered/malformed suffix** (`"Alias creation time is expired, please retry"`) — `:73` / `:188`. Every non-valid token funnels here: `check_suffix_signature()` (`app/alias_suffix.py:37-42`) wraps `signer.unsign(..., max_age=600)` in `except itsdangerous.BadSignature: return None`, and in **itsdangerous 1.1.0** `SignatureExpired ⊂ BadTimeSignature ⊂ BadSignature`, so an expired, tampered, *or* malformed token all yield `None` — which the handler reports as `412` via the `if not alias_suffix:` branch (`:71-73` / `:186-188`). (Runtime-verified: tampered, `garbage.garbage.garbage`, and empty suffixes all return `412`.)
- **`400`** for the remaining validation failures: quota (`:55`), empty body (`:62` / `:151`), wrong body format (`:154`), bad prefix (`:168`), bad `mailbox_ids`/mailbox (`:172` / `:177` / `:181`), wrong prefix/suffix (`:79` / `:194`), consecutive dots (`:92-93` / `:207-208`), and the v3 quota/format branch (`:144`).
- **Dead code — the `400 "Tampered suffix"` branch never fires on the API.** Both handlers wrap the suffix check in `try: … except Exception: return jsonify(error="Tampered suffix"), 400` (`:74-76` / `:189-191`), but this `except` is **unreachable for any API-supplied input**. The API always coerces `signed_suffix` to a *string* (`:65` for v2; `:157-158` for v3), and — as shown in the `412` bullet above — `check_suffix_signature()` catches `itsdangerous.BadSignature` internally and **returns `None` rather than raising** for any string token (expired, tampered, *or* malformed). Control therefore always takes the `if not alias_suffix:` → `412` path, never this `400` branch. **Verified at runtime:** tampered (signature-tail flip and value-portion tamper), `garbage.garbage.garbage`, and empty-string suffixes each returned **`412`** with `{"error":"Alias creation time is expired, please retry"}` — **none returned `400`**. (The `400 "Tampered suffix"` string is thus reachable only as documentation of intent, not as an observable response on these endpoints.)

As with the dashboard, all of these return before the alias is committed, so a rejected API request leaves the database unchanged.

### (d) Rate-limit / concurrency → `429` (three independent mechanisms)

Three independent layers can each yield an HTTP `429`. They do **not** all behave the same way when Redis is unavailable: the **two custom Redis-backed mechanisms** (the token bucket inside `Alias.create()` and the `parallel_limiter` NX lock) degrade gracefully to no-ops, whereas **Flask-Limiter remains active** regardless of Redis and is switched off only by an explicit `DISABLE_RATE_LIMIT` flag. The per-layer behavior is detailed below.

1. **Flask-Limiter decorator** — `@limiter.limit(ALIAS_LIMIT, methods=["POST"])` on the dashboard view, with `ALIAS_LIMIT = "100/day;50/hour;5/minute"` (`app/config.py:448`). The limiter is constructed at `app/extensions.py:23` (`Limiter(key_func=...)`) and wired in via `limiter.init_app(app)` at `server.py:167`. Its **storage** is Redis **only when `MEM_STORE_URI` is configured** — `app.config[...STORAGE_URL] = MEM_STORE_URI` at `server.py:163-165` (`MEM_STORE_URI` defaults to `None`, `app/config.py:568`); when it is unset, Flask-Limiter falls back to its **default in-memory storage and still enforces the limit**. So, unlike the two custom mechanisms below, it does **not** no-op when Redis is absent — it is disabled wholesale **only** when `config.DISABLE_RATE_LIMIT` is set, via the request filter at `app/extensions.py:26-28` (flag at `app/config.py:602`). Exceeding the window → `429`.
2. **Redis token-bucket inside `Alias.create()`** — `rate_limiter.check_bucket_limit(...)` (`app/rate_limiter.py:19-42`), driven by the per-tier limits `ALIAS_CREATE_RATE_LIMIT_FREE = "10,900:50,3600"` (`app/config.py:554-556`) and `ALIAS_CREATE_RATE_LIMIT_PAID = "50,900:200,3600"` (`app/config.py:557-559`). On breach it logs `LOG.i("Rate limit hit for ...")` (`app/rate_limiter.py:33-35`), records a `BucketRateLimit` New Relic event (`:36-39`), and raises `werkzeug.exceptions.TooManyRequests()` (`:40`). If Redis is missing it no-ops immediately (`if not lock_redis: return`, `app/rate_limiter.py:28-29`); if Redis errors it logs `LOG.e("Cannot connect to redis")` and continues (`:41-42`).
3. **`parallel_limiter` Redis NX lock** — `@parallel_limiter.lock(name="alias_creation")`. `acquire_lock()` does a Redis `set(name, value, ex=timedelta(seconds=max_wait_secs), nx=True)` and raises `exceptions.TooManyRequests()` if the key already exists (`app/parallel_limiter.py:30-34`); `max_wait_secs` defaults to `5` (`app/parallel_limiter.py:23`), so it blocks a second concurrent same-user creation within a 5-second window. It no-ops when `lock_redis` is `None` (`app/parallel_limiter.py:51-52`).

### Why / Reasoning

The design fails **fast and safe**. Validation rejections (category a/c) all return *before* any row is staged, so a bad request can never leave a partial write. The `IntegrityError` path (category b) is the proof of transactional integrity: because `Alias.create()` does not commit on its own (it defaults `commit`/`flush` to `False` at `app/models.py:1629-1630`) and the view owns the single commit (`app/dashboard/views/custom_alias.py:158`), a constraint violation rolls the *entire* staged unit of work back (`:148`), leaving no orphaned alias, metric, or audit row. The three `429` layers exist at deliberately different scopes — an HTTP-window limit (per IP/user), a per-user creation token bucket inside the model, and a short concurrency lock. The **two custom Redis-backed layers** (token bucket and parallel lock) are intentionally **best-effort**: they no-op when Redis is unavailable so a developer without Redis is never blocked. **Flask-Limiter is different** — it is always active (using in-memory storage when `MEM_STORE_URI` is unset) and is bypassed only by the explicit `DISABLE_RATE_LIMIT` flag — so the "Redis-optional" property applies to the two custom mechanisms, not to the HTTP-window limiter.


---

## Consolidated failure-mode matrix

| Scenario | Trigger (code locator) | API / HTTP surface | Log line | Resulting DB state |
|----------|------------------------|--------------------|----------|--------------------|
| Quota exceeded (dashboard) | `app/dashboard/views/custom_alias.py:36-42` | `302` redirect + warning flash | `LOG.d("%s can't create new alias")` | Unchanged (no write) |
| Invalid CSRF (dashboard) | `app/dashboard/views/custom_alias.py:56-58` | `302` redirect + flash | — | Unchanged |
| Bad prefix (dashboard) | `app/dashboard/views/custom_alias.py:64`; `app/alias_utils.py:418-425` | `302` redirect + error flash | — | Unchanged |
| Bad/unverified mailbox (dashboard) | `app/dashboard/views/custom_alias.py:74-87` | `302` redirect + flash | — | Unchanged |
| Expired suffix (dashboard) | `app/dashboard/views/custom_alias.py:90-94` | `302` redirect + flash | `LOG.w("Alias creation time expired for %s")` | Unchanged |
| Tampered suffix (dashboard) | `app/dashboard/views/custom_alias.py:95-98` | `302` redirect + flash | `LOG.w("Alias suffix is tampered, user %s")` | Unchanged |
| Consecutive dots (dashboard) | `app/dashboard/views/custom_alias.py:104-105` | `302` redirect + flash | — | Unchanged |
| Invalid email (dashboard) | `app/dashboard/views/custom_alias.py:108-113` | `302` redirect + flash | — | Unchanged |
| Duplicate / trash (pre-insert, dashboard) | `app/dashboard/views/custom_alias.py:117-135` (flash without `return`, falls through to `render_template` `:166`) | **`200`** — inline `render_template` re-render + error flash (no redirect) | — | Unchanged (no insert) |
| Duplicate race at INSERT (dashboard) | `app/dashboard/views/custom_alias.py:146-150` | `302` redirect + error flash | `LOG.w("Alias %s already exists")` | **Rolled back — no partial row** (`Session.rollback()` `:148`) |
| Duplicate (API) | `app/api/views/new_custom_alias.py:88` / `:203` | `409` JSON error | — | Unchanged |
| Expired / tampered / malformed suffix (API) | `app/api/views/new_custom_alias.py:71-73` / `:186-188` (via `check_suffix_signature` → `None`, `app/alias_suffix.py:37-42`) | `412` JSON error (`"Alias creation time is expired, please retry"`) | `LOG.w("Alias creation time expired for %s")` | Unchanged |
| Validation errors (API) | `app/api/views/new_custom_alias.py:55`, `:62`/`:151`, `:154`, `:168`, `:79`/`:172`/`:177`/`:181`, `:92-93`/`:207-208`, `:144` | `400` JSON error | — | Unchanged |
| Tampered / malformed suffix — *documented-but-dead* `400` branch (API) | `app/api/views/new_custom_alias.py:74-76` / `:189-191` (`except Exception`) | **Never fires** — surfaces as `412` (see the "Expired / tampered / malformed suffix (API)" row); `check_suffix_signature` returns `None`, not an exception | `LOG.w("Alias suffix is tampered, user %s")` *(would-be)* | Unchanged |
| HTTP-window rate limit | `@limiter.limit(ALIAS_LIMIT)`; `app/config.py:448`; `app/extensions.py:23` | `429` | (Flask-Limiter) | Unchanged |
| Per-user creation bucket | `app/rate_limiter.py:19-42`; limits `app/config.py:554-559` | `429` (`TooManyRequests` `:40`) | `LOG.i("Rate limit hit for ...")` | Unchanged |
| Concurrent same-user creation | `app/parallel_limiter.py:30-34` (5s window, `:23`) | `429` (`TooManyRequests` `:34`) | — | Unchanged |
| Redis unavailable | `app/rate_limiter.py:28-29`, `:41-42`; `app/parallel_limiter.py:51-52` | **Two custom limiters no-op**; Flask-Limiter stays active using in-memory storage (`server.py:163-165`; `app/config.py:568`) | `LOG.e("Cannot connect to redis")` (on error) | Normal creation proceeds (custom limiters skipped) |

---

## Summary of key findings

1. **Two response shapes for four flows.** The web dashboard custom-alias flow is `POST /dashboard/custom_alias` with an `application/x-www-form-urlencoded` body (`app/dashboard/views/custom_alias.py:30`, fields at `:59-62`) and responds with **`302` redirect + flash** (`:159-161`). The REST API flows respond with **`201` + JSON** (`app/api/views/new_custom_alias.py:110-111` / `:233-234`; `app/api/views/new_random_alias.py:115-116`), the body assembled from `serialize_alias_info_v2` (`app/api/serializer.py:55-92`) plus an endpoint-injected top-level `alias` key, with `latest_activity: null` for a brand-new alias (`app/api/serializer.py:77`).
2. **The normal case writes exactly three tables: `alias`, `alias_audit_log`, and `daily_metric`** — from the three unconditional statements in `Alias.create()` (`app/models.py:1660`, `:1661`, `:1688-1690`) plus the view's single commit (`app/dashboard/views/custom_alias.py:158`). Extra mailboxes add `alias_mailbox` rows (`:152-156`); partner/API/hostname variants add `users`/`api_key`/`alias_used_on`; and `sync_event` is gated off in dev.
3. **Audit-log and metric are in-transaction; events are gated off in dev.** The `alias_audit_log` and `daily_metric` writes are committed atomically with the alias, while the `AliasCreated` event is a logging no-op in standard dev because the dispatcher is triple-gated (`app/events/event_dispatcher.py:57-70`) and `EVENT_WEBHOOK` defaults to `None` (`app/config.py:612`) with `john@wick.com` having no partner user — so **no `sync_event` row and no `NOTIFY`**. The background workers `event_listener.py`, `job_runner.py`, and `cron.py` live at the **repository root** and are not part of — nor synchronously triggered by — a normal alias creation (though `job_runner.py` does contain a separate `JOB_SEND_ALIAS_CREATION_EVENTS` job, `job_runner.py:295-302`, that is enqueued only by the Proton initial-alias-sync path, `app/proton/proton_callback_handler.py:46-52`).
4. **Failures fail-fast or roll back, never leaving a partial row.** Validation errors surface before any row is written (dashboard: error flash + a `302` redirect for most branches, or an inline `200` re-render for the duplicate/trash pre-insert checks; API `400`/`409`/`412`), and a duplicate race at INSERT triggers `Session.rollback()` (`app/dashboard/views/custom_alias.py:146-150`), discarding the entire staged unit of work. Three independent `429` limiters guard the path at different scopes: the Flask-Limiter HTTP-window decorator (`app/config.py:448`; always active unless `DISABLE_RATE_LIMIT`, `app/extensions.py:26-28`), plus two **Redis-optional** custom mechanisms — the per-user token bucket (`app/rate_limiter.py:19-42`) and the concurrency NX lock (`app/parallel_limiter.py:30-34`) — that no-op when Redis is unavailable.

