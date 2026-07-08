# What actually happens when SimpleLogin starts locally in development

> **Binding acceptance criterion (verbatim from the request):**
> *"I'm not looking for what the code says should happen, I want to know what actually gets printed when you run it."*

This document answers, from **observed runtime output**, what really happens when the SimpleLogin
backend is started in a local development environment. It is organized around the three axes that
were asked about:

1. **Startup in dev mode** — how the backend process starts, how configuration is loaded, which log
   messages signal readiness, and which ports/endpoints are exposed.
2. **Authenticated request handling** — where a request first enters the app, how the user's identity
   is determined at runtime, and how that authentication context is carried through the request
   (this is the primary concern).
3. **Background work** — whether starting the app auto-starts any background jobs or schedulers, or
   whether anything runs beyond handling incoming HTTP requests.

**Evidence conventions used throughout.** Every behavioral claim is paired with (a) the actual,
complete output that was captured, and (b) a `file:line` (or config-key) reference to the code that
produces it. Command lines that produced each block of output are shown immediately above the
output. Output is reproduced verbatim, with **one** deliberate exception for safety: the opaque
signed value of the `slapp` session cookie is masked as `<SIGNED_VALUE_REDACTED>` wherever it appears
in an HTTP response head (the masking is performed by the very command shown, so each block is
exactly reproducible; all other bytes — status lines, every header name, real `Content-Length`
values, and all cookie *attributes* — are verbatim). Blocks that contain no such secret (console
startup output, server-side log lines, `psql`/version output) are shown fully unedited and are
labeled as such. Anything that was *not* directly observed at runtime is explicitly labeled
**_inferred_**.

## Runtime under test

| Property | Value | How it was confirmed |
|----------|-------|----------------------|
| Python | 3.10.18 | `Server:` header `Python/3.10.18` observed on every HTTP response (see §1.5) |
| OS/host | Canonical Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | provided runtime |
| Code commit | `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` | `git rev-parse HEAD` |
| PostgreSQL | 15, `localhost:5432`, DB `simplelogin` | `DB_URI` in `.env`; 77 public tables after migration |
| Redis | `localhost:6379` (replies `PONG`) | `redis-cli ping` |
| Flask | 1.1.2 | pin in `poetry.lock`; banner + behavior in §1.4 |
| Werkzeug | 1.0.1 | `Server:` header `Werkzeug/1.0.1` observed (see §1.5) |
| Flask-Login | 0.5.0 | `importlib.metadata` probe below; identity flow in §2 |
| SQLAlchemy | 1.3.24 | `importlib.metadata` probe below |
| Alembic | 1.4.3 | `importlib.metadata` probe below; `alembic upgrade head` used for schema |
| python-dotenv | 0.14.0 | `importlib.metadata` probe below; loads `.env` in `app/config.py:L9` |

### How the runtime-under-test values were confirmed (commands + complete output)

Each value in the table above was captured directly inside the canonical container. The exact
commands and their complete, unedited output are below:

```text
$ git rev-parse HEAD
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c

$ redis-cli ping
PONG

$ python --version
Python 3.10.18

$ PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin -tA \
    -c "SHOW server_version;" \
    -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
15.13 (Debian 15.13-0+deb12u1)
77

$ python -c "import importlib.metadata as m
for dist in ['flask','werkzeug','flask-login','sqlalchemy','alembic','python-dotenv']:
    print(f'{dist}=={m.version(dist)}')"
flask==1.1.2
werkzeug==1.0.1
flask-login==0.5.0
sqlalchemy==1.3.24
alembic==1.4.3
python-dotenv==0.14.0
```

The `Flask 1.1.2` and `Werkzeug 1.0.1` rows are additionally corroborated at runtime by the
`Server: Werkzeug/1.0.1 Python/3.10.18` response header in §1.5; the table's `PostgreSQL 15` is the
major version of the `15.13` shown here.

## Exact build / invocation commands used

These mirror the canonical developer workflow in `CONTRIBUTING.md:L106` ("## Run the code locally"
is at `CONTRIBUTING.md:L77`):

```bash
# 1. Configuration: canonical .env from the template
cp example.env .env
#    -> URL=http://localhost:7777            (example.env:L6)
#    -> DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin (example.env:L75)
#    -> FLASK_SECRET=secret                  (example.env:L77)
#    -> LOCAL_FILE_UPLOAD=true               (example.env:L136)

# 2. Services
service postgresql start && service redis-server start

# 3. Schema + seed data (mirrors CONTRIBUTING.md:L106)
alembic upgrade head        # applies the Alembic revisions under migrations/versions/
flask dummy-data            # seeds dev users john@wick.com and winston@continental.com

# 4. Start the dev server (the object of this investigation)
python server.py            # == python3 server.py in CONTRIBUTING.md:L106
```

Dev login credentials used for the authenticated-request axis are the documented ones:
`john@wick.com / password` (`CONTRIBUTING.md:L109`).

### Environment-only build caveats (reproducibility notes — repository manifests were NOT changed)

Two lock-pinned packages needed local build workarounds while provisioning the environment. **These
are local-environment notes only. `pyproject.toml` and `poetry.lock` were not modified, and neither
caveat affects the observed startup / request / background behavior below.**

- **`pyre2 0.3.6`** has no published CPython 3.10 wheel, so it was compiled from source. That build
  needs the system packages `libre2-dev` and `ninja-build` — the same system dependencies already
  declared in the project `Dockerfile`.
- **`cbor2 5.2.0`** (the locked pin) ships an sdist with broken metadata that installers reject, so
  `cbor2 5.4.6` was substituted **in the local environment only**.

---

# Section 1 — Startup in dev mode

## 1.1 Process launch and entry point

Running `python server.py` executes the module top-to-bottom. At the bottom, under the
`if __name__ == "__main__":` guard, it starts the Werkzeug/Flask development server:

- `app.run(debug=True, port=7777)` — **`server.py:L588`**.

The Flask application object is assembled by the application-factory function:

- `def create_app() -> Flask:` — **`server.py:L139`**.

Inside the factory, the WSGI app is wrapped by `ProxyFix` (so `X-Forwarded-*` headers from a
reverse proxy are honored):

- `app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_host=1)` — **`server.py:L142`**.

and all blueprints are attached by:

- `def register_blueprints(app: Flask):` — **`server.py:L233`** (the individual
  `app.register_blueprint(...)` calls are at `server.py:L234-L246`; enumerated in §1.5).

The production WSGI entry point re-uses the very same factory — `wsgi.py` is only
`from server import create_app` (`wsgi.py:L1`) and `app = create_app()` (`wsgi.py:L3`). **_Inferred_
for the production path:** in dev we run `server.py` (not `wsgi.py`), so the production reuse of
`create_app()` via `wsgi.py` is read from source, not observed here (see §3.3 for the production
contrast).

_The dev-server facts above — `app.run(...)`, `create_app()`, `ProxyFix`, and
`register_blueprints()` — are each corroborated at runtime below by the banner, the `Server:` header,
and the reachable endpoint surface._

## 1.2 Configuration loading

Configuration is twelve-factor style: environment variables loaded from a `.env` file by
`python-dotenv`. In `app/config.py`:

- `from dotenv import load_dotenv` — **`app/config.py:L9`**.
- The loader first checks an optional `CONFIG` env var — `config_file = os.environ.get("CONFIG")`
  (**`app/config.py:L65`**) — and, when it is unset (the canonical dev case), falls back to
  `load_dotenv()` which reads the default `.env` — **`app/config.py:L71`**.
- Required variables are read at *import time*, e.g. `URL = os.environ["URL"]`
  (**`app/config.py:L79`**) and `DB_URI = os.environ["DB_URI"]` (**`app/config.py:L192`**).

The **first five lines of startup console output originate here** — they are plain `print(...)`
calls executed while `app/config.py` is imported:

| Observed line | Emitting statement |
|---------------|--------------------|
| `>>> URL: http://localhost:7777` | `print(">>> URL:", URL)` — `app/config.py:L80` |
| `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` | `app/config.py:L123` |
| `Paddle param not set` | `print("Paddle param not set")` — `app/config.py:L217` |
| `WARNING: Use a temp directory for GNUPGHOME /tmp/<random>` | GNUPGHOME temp-dir warning — `app/config.py:L262` |
| `Upload files to local dir` | `print("Upload files to local dir")` — `app/config.py:L328` |

The `Upload files to local dir` branch is taken because `LOCAL_FILE_UPLOAD=true`
(`example.env:L136`). The GNUPGHOME temp-dir path is randomly generated per process, which is why it
differs between the two startup blocks in §1.4.

## 1.3 Logging initialization and readiness markers

Logging is initialized in `app/log.py`:

- It prints the init marker: `print(">>> init logging <<<")` — **`app/log.py:L67`**.
- It installs a **stdout** stream handler: `logging.StreamHandler(sys.stdout)` — **`app/log.py:L41`**.
- It **disables the Werkzeug logger** — `log = logging.getLogger("werkzeug")` (**`app/log.py:L70`**)
  then `log.disabled = True` (**`app/log.py:L71`**). *This single fact explains the two most
  surprising observations in §1.4/§1.5: the absence of Werkzeug's `* Running on ...` line and the
  absence of the default per-request access logs.*
- It defines the `LOG` object and its shortcuts: `LOG = _get_logger("SL")` (**`app/log.py:L79`**);
  the `.d/.i/.w/.e` aliases are wired at `app/log.py:L74-L77` (`.d`=debug L74, `.i`=info L75,
  `.w`=warning L76, `.e`=exception L77).

The single `SL - DEBUG` line that appears at the end of each startup block is emitted while
`app/utils.py` is imported and loads the word list:

- `LOG.d("load words file: %s", WORDS_FILE_PATH)` — **`app/utils.py:L17`** (the `open(...)` is at
  `app/utils.py:L16`).

Notably, the runtime log line **self-cites its own source location** —
`"/app/app/utils.py:17" - <module>()` — which is direct runtime proof that this line comes from
`app/utils.py:L17`.

**Empirical readiness signal.** Because the Werkzeug logger is disabled, the customary
`* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)` line never appears. The real,
observed "it's up" signal is the Flask banner block ending in `* Debug mode: on`, preceded by the
`>>> URL:` and `>>> init logging <<<` markers (see §1.4). Actual request-serving readiness was then
confirmed independently by a successful `curl` handshake (§1.5).

## 1.4 Verbatim startup output (primary evidence)

Command that produced the output (run from the repository root `/app`):

```bash
python server.py
```

The **complete, unedited** console output captured from that run is below. It is reproduced exactly
as printed (real timestamps, PIDs, and random GNUPGHOME temp-dir names preserved). **The startup
prints appear twice** — see nuance C1 in the Critical Runtime Nuances section for why:

```text
/app/venv/lib/python3.10/site-packages/flask_limiter/errors.py:4: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  from pkg_resources import get_distribution
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/warewvmhiagftxcpvfcp
Upload files to local dir
>>> init logging <<<
2026-07-08 05:19:16,191 - SL - DEBUG - 15160 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
/app/venv/lib/python3.10/site-packages/flask_limiter/errors.py:4: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  from pkg_resources import get_distribution
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/oddxbzsvhqerfnumtlzo
Upload files to local dir
>>> init logging <<<
2026-07-08 05:19:18,046 - SL - DEBUG - 15181 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

Line-by-line, what each observed line is and where it comes from:

- The two leading `UserWarning`/`from pkg_resources import get_distribution` lines are emitted by the
  third-party `flask_limiter` package at import (`flask_limiter/errors.py:4`), not by SimpleLogin
  code. They are reported here because they are part of "what actually gets printed."
- `>>> URL: http://localhost:7777` → `app/config.py:L80` (value from `URL` = `example.env:L6`).
- `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` → `app/config.py:L123`.
- `Paddle param not set` → `app/config.py:L217`.
- `WARNING: Use a temp directory for GNUPGHOME /tmp/warewvmhiagftxcpvfcp` (and, in the second block,
  `/tmp/oddxbzsvhqerfnumtlzo`) → `app/config.py:L262`. The temp-dir name is random per process, so
  the two blocks differ here.
- `Upload files to local dir` → `app/config.py:L328` (taken because `LOCAL_FILE_UPLOAD=true`).
- `>>> init logging <<<` → `app/log.py:L67`.
- `... SL - DEBUG - <pid> - "/app/app/utils.py:17" - <module>() - ... - load words file: /app/local_data/test_words.txt`
  → `app/utils.py:L17` (self-cited by the runtime line itself).
- The four-line Flask banner (`* Serving Flask app "server" (lazy loading)` … `* Debug mode: on`) is
  Flask 1.1.2's own banner, printed once (only in the first/supervisor block — see C1).

**Two process IDs are visible in the output** — `15160` in the first block and `15181` in the
second. This is the reloader parent/child split (proof in C1).

## 1.5 Ports and endpoints

**Bound address: `127.0.0.1:7777`.** This is `app.run(debug=True, port=7777)` at
**`server.py:L588`** (host defaults to `127.0.0.1`). It was confirmed empirically with `curl`
(the authoritative check — see the caveat below):

Command (headers only; the `sed` masks the opaque signed session-cookie value for safety):

```bash
curl -sD - -o /dev/null http://127.0.0.1:7777/auth/login \
  | sed -E 's/(slapp=)[^;]+/\1<SIGNED_VALUE_REDACTED>/'
```

Complete response head as produced. Only the opaque signed `slapp` cookie value is masked (by the
`sed` above); every other byte is verbatim. It is therefore deliberately **not** labeled "unedited":
the single `<SIGNED_VALUE_REDACTED>` token marks the one redacted field, while all header names, the
real `Content-Length`, and every cookie attribute (`Expires`, `HttpOnly`, `Path`, `SameSite`) are
shown exactly as returned:

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 221503
Vary: Cookie
Set-Cookie: slapp=<SIGNED_VALUE_REDACTED>; Expires=Wed, 15-Jul-2026 05:20:22 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 08 Jul 2026 05:20:22 GMT
```

(The login page is rendered per request — it embeds a fresh CSRF token — so `Content-Length` varies
by a few hundred bytes between calls; `221503` is the value from this exact capture.) The
`Server: Werkzeug/1.0.1 Python/3.10.18` header is direct runtime confirmation of both the
Werkzeug 1.0.1 and Python 3.10.18 pins. A bare `GET /` (no session) returns a redirect to the login
page:

Command (same masking convention):

```bash
curl -sD - -o /dev/null http://127.0.0.1:7777/ \
  | sed -E 's/(slapp=)[^;]+/\1<SIGNED_VALUE_REDACTED>/'
```

Complete response head as produced (only the signed `slapp` value masked; all else verbatim):

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://127.0.0.1:7777/auth/login
Vary: Cookie
Set-Cookie: slapp=<SIGNED_VALUE_REDACTED>; Expires=Wed, 15-Jul-2026 05:20:22 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 08 Jul 2026 05:20:22 GMT
```

> **Caveat (labeled).** Socket-table tools (`ss -ltnp`, `lsof`) produced **no output** for port 7777
> inside this container — a false negative caused by the container not exposing its socket table to
> those tools, **not** evidence that nothing is listening. The successful `curl` TCP handshake above
> is authoritative and proves the server is bound and serving on `127.0.0.1:7777`.

**Endpoint surface.** The application factory makes 11 blueprint registrations via
`register_blueprints()` (`server.py:L233-L246`) — 10 distinct blueprints, since `oauth_bp` is mounted
twice. Each URL prefix below was read from the blueprint's own definition (or, for `oauth_bp`, from
the explicit override in `register_blueprints()`):

| Blueprint (registration) | URL prefix | `file:line` of the prefix |
|--------------------------|------------|---------------------------|
| `auth_bp` | `/auth` | `app/auth/base.py:L4` |
| `monitor_bp` | `/` (root; serves `/git`, `/live`, `/exception`) | `app/monitor/base.py:L3` |
| `dashboard_bp` | `/dashboard` | `app/dashboard/base.py:L6` |
| `developer_bp` | `/developer` | `app/developer/base.py:L6` |
| `phone_bp` | `/phone` | `app/phone/base.py:L6` |
| `oauth_bp` | `/oauth` | override at `server.py:L240` |
| `oauth_bp` | `/oauth2` (same blueprint mounted a 2nd time) | override at `server.py:L241` |
| `onboarding_bp` | `/onboarding` | `app/onboarding/base.py:L6` |
| `discover_bp` | `/discover` | `app/discover/base.py:L6` |
| `internal_bp` | `/internal` | `app/internal/base.py:L6` |
| `api_bp` | `/api` | `app/api/base.py:L11` |

> Note: `monitor_bp` is mounted at `/` (not `/monitor`); its actual endpoints are `/git`, `/live`,
> and `/exception` (`app/monitor/views.py:L5,L10,L15`).

Beyond the blueprints, a handful of routes are attached **directly on the app** inside
`create_app()`, most importantly the **bare root `/`** (see §2.1 — it is an app-level `index`
endpoint, not a blueprint route) and `/health` (`server.py:L213`).

The full endpoint surface was **enumerated at runtime** with a temporary probe kept **outside** the
repository tree (`/tmp/enum_routes.py`). It builds the app via the same `create_app()` factory the
dev server uses and reads `app.blueprints` and `app.url_map`. Command and complete captured output:

```bash
python /tmp/enum_routes.py
```

```text
=== ENUM_RESULT_START ===
blueprint_count: 25
url_rule_count: 292
blueprints: admin, adminauditlog, alias, api, auth, coupon, customdomain, dailymetric, dashboard, developer, discover, email_search, internal, invalidmailboxdomain, mailbox, manualsubscription, metric2, monitor, newsletter, newsletteruser, oauth, onboarding, phone, providercomplaint, user
representative_route /auth/login          present=True
representative_route /dashboard/          present=True
representative_route /api/user_info       present=True
representative_route /                    present=True
representative_route /oauth/authorize     present=True
representative_route /oauth2/authorize    present=True
=== ENUM_RESULT_END ===
```

So the *live* app exposes **25 blueprints** and **292 URL rules**. Ten of those blueprints come from
`register_blueprints()` (the table above lists **11 registrations** because `oauth_bp` is mounted
twice, at `/oauth` and `/oauth2`, but it is a single blueprint named `oauth`). The remaining **15** —
`admin`, `adminauditlog`, `alias`, `coupon`, `customdomain`, `dailymetric`, `email_search`,
`invalidmailboxdomain`, `mailbox`, `manualsubscription`, `metric2`, `newsletter`, `newsletteruser`,
`providercomplaint`, `user` — are registered by **Flask-Admin** at runtime (the admin UI), not by
`register_blueprints()`. All six representative routes are confirmed present (`present=True`):
`/auth/login`, `/dashboard/`, `/api/user_info`, `/`, `/oauth/authorize`, `/oauth2/authorize`.

---

# Section 2 — Authenticated request handling (the primary concern)

This section answers three named sub-questions: **where the request enters the app**, **how the
user identity is determined at runtime**, and **how the auth context is carried through the
request**. Each claim is backed by the server-side log lines captured while a real login was driven
through the actual `/auth/login` entry point (not a debug bypass).

## 2.1 Entry point and per-request hooks

An HTTP request enters the WSGI application built by `create_app()` (`server.py:L139`) and wrapped by
`ProxyFix` (`server.py:L142`); Flask then routes it to the matching view — either a blueprint view or
one of the routes attached directly on the app inside `create_app()`. The clearest example of the
latter is the **root `/` `index` view**, defined by `set_index_page(app)` (called at
`server.py:L173`): `@app.route("/", methods=["GET", "POST"])` (`server.py:L250`), `def index():`
(`server.py:L251`). This view reads `current_user` directly and branches on it:
`if current_user.is_authenticated:` (`server.py:L252`) → `redirect(url_for("dashboard.index"))`
(`server.py:L253`), `else:` (`server.py:L254`) → `redirect(url_for("auth.login"))`
(`server.py:L255`). That branch is exactly the observed behavior in §2.4 (authenticated `GET /` →
`302 /dashboard/`; unauthenticated `GET /` → `302 /auth/login`).

Two application-wide hooks bracket every request:

- **`before_request`** — `@app.before_request` (`server.py:L257`), `def before_request():`
  (`server.py:L258`). It seeds timing and referral state: `g.start_time = time.time()`
  (`server.py:L265`) and, when a `?slref=` is present, `session["slref"] = ref_code`
  (`server.py:L270`).
- **`after_request`** — `@app.after_request` (`server.py:L272`), `def after_request(res):`
  (`server.py:L273`). It emits the **one per-request log line** the app produces, via `LOG.d(...)` at
  **`server.py:L284`**, using the format string `"%s %s %s %s %s, takes %s"` (`server.py:L285`) =
  `remote_addr, method, path, args, status_code, elapsed`.

That `after_request` line is the observable "a request was handled" signal. Captured examples
(unedited; each self-cites `"/app/server.py:284" - after_request()`):

```text
2026-07-08 05:19:50,252 - SL - DEBUG - 15181 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.02341294288635254
2026-07-08 05:19:50,262 - SL - DEBUG - 15181 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0006802082061767578
```

## 2.2 Runtime user identity — keyed on `alternative_id` (a UUID), not the primary key

This is the direct answer to *"how the user identity is determined at runtime."*

The `User` model mixes in Flask-Login's `UserMixin`:

- `class User(Base, ModelMixin, UserMixin, PasswordOracle):` — **`app/models.py:L336`**.

It has a dedicated identity column, distinct from the integer primary key:

- `alternative_id = sa.Column(sa.String(128), unique=True, nullable=True)` — **`app/models.py:L482`**.

Crucially, Flask-Login asks the model for its session identity via `get_id()`, and SimpleLogin
returns the **`alternative_id`**, not the primary key:

- `def get_id(self):` (`app/models.py:L595`) → `if self.alternative_id:` (`app/models.py:L596`) →
  `return self.alternative_id` (`app/models.py:L597`).

On each subsequent request, Flask-Login reverses that value back into a `User` via the registered
user-loader:

- `@login_manager.user_loader` (`server.py:L220`), `def load_user(alternative_id):`
  (`server.py:L221`), `user = User.get_by(alternative_id=alternative_id)` (`server.py:L222`).

The `LoginManager` itself is created in `app/extensions.py`:

- `login_manager = LoginManager()` (`app/extensions.py:L7`) with
  `login_manager.session_protection = "strong"` (`app/extensions.py:L8`), and it is wired into the
  app by `login_manager.init_app(app)` (`server.py:L438`).

**Runtime proof (observed).** The `alternative_id` was read straight from the database, and the
signed `slapp` cookie was decoded with the app's own `FLASK_SECRET` via Flask's
`SecureCookieSessionInterface` (in the same `/tmp/probe_auth.py` probe used in §2.4).

The database row for `john@wick.com`:

```text
$ PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin \
    -c "SELECT id, email, alternative_id FROM users WHERE email='john@wick.com';"
 id |     email     |            alternative_id
----+---------------+--------------------------------------
  1 | john@wick.com | fc14048c-8214-4d7a-a559-31c0df0775cd
(1 row)
```

The decoded `slapp` payload captured by that probe (the full block is reproduced verbatim in §2.4
under "SESSION COOKIE DECODE"):

```text
decoded payload keys: ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
_user_id: fc14048c-8214-4d7a-a559-31c0df0775cd
```

The session's `_user_id` equals the **`alternative_id` UUID**, **not** the primary key `1`. That is
exactly the value `get_id()` returns (`app/models.py:L597`) and the value `load_user()` reverses
(`server.py:L222`). So runtime identity is keyed on the `alternative_id` UUID.

## 2.3 The login path exercised (the real entry point)

The `/auth/login` view lives in `app/auth/views/login.py`:

- `@auth_bp.route("/login", methods=["GET", "POST"])` (`app/auth/views/login.py:L21`),
  rate-limited by `@limiter.limit(...)` (`app/auth/views/login.py:L22`), `def login():`
  (`app/auth/views/login.py:L25`).
- On POST it validates the password: `if not user or not user.check_password(form.password.data):`
  (`app/auth/views/login.py:L45`) and, on success, hands off with
  `return after_login(user, next_url)` (`app/auth/views/login.py:L72`).

`after_login()` performs the actual Flask-Login sign-in, in `app/auth/views/login_utils.py`:

- `def after_login(user, next_url, login_from_proton: bool = False):`
  (`app/auth/views/login_utils.py:L12`),
- logs `LOG.d("log user %s in", user)` (`app/auth/views/login_utils.py:L35`),
- calls `login_user(user)` (`app/auth/views/login_utils.py:L36`) — this is what writes
  `_user_id = get_id()` into the session cookie,
- and (no `next_url`) redirects to the dashboard after `LOG.d("redirect user to dashboard")`
  (`app/auth/views/login_utils.py:L44`).

An optional identity probe endpoint also exists — the API `/user_info`:
`@api_bp.route("/user_info")` (`app/api/views/user_info.py:L50`), `@require_api_auth`
(`app/api/views/user_info.py:L51`), `def user_info():` (`app/api/views/user_info.py:L52`). Identity
resolution here was instead proven directly by decoding the session cookie (§2.2), which is a
stronger, more direct demonstration that `_user_id == alternative_id`.

## 2.4 Observed authentication evidence

The flow was driven by a temporary HTTP client kept **outside** the repository tree
(`/tmp/probe_auth.py`, using `requests`). It exercises the **real** `/auth/login` entry point — a
`GET` to fetch the login form and its CSRF token, a `POST` with the seeded credentials
`john@wick.com / password` plus that token, then authenticated and fresh-session follow-ups — all
with `allow_redirects=False`, so each individual response (including every `302` and its `Location`)
is observed directly. The opaque signed `slapp` value is masked by the probe for safety; every other
byte of each response head is verbatim as returned by the server.

Command:

```bash
python /tmp/probe_auth.py
```

Complete captured client-side output — all five conditions plus the session-cookie decode, exactly
as printed:

```text
========================================================================
CONDITION 1  GET /auth/login   (no prior session)
------------------------------------------------------------------------
request : GET http://127.0.0.1:7777/auth/login
status  : HTTP/1.0 200 OK
response headers:
    Content-Type: text/html; charset=utf-8
    Content-Length: 221663
    Vary: Cookie
    Set-Cookie: slapp=<SIGNED_VALUE_REDACTED>; Expires=Wed, 15-Jul-2026 05:21:35 GMT; HttpOnly; Path=/; SameSite=Lax
    Server: Werkzeug/1.0.1 Python/3.10.18
    Date: Wed, 08 Jul 2026 05:21:35 GMT
csrf_token parsed from login form: present=True length=91
body bytes: 221663

========================================================================
CONDITION 2  POST /auth/login  (john@wick.com / password + csrf_token)
------------------------------------------------------------------------
request : POST http://127.0.0.1:7777/auth/login
status  : HTTP/1.0 302 FOUND
response headers:
    Content-Type: text/html; charset=utf-8
    Content-Length: 229
    Location: http://127.0.0.1:7777/dashboard/
    Vary: Cookie
    Set-Cookie: slapp=<SIGNED_VALUE_REDACTED>; Expires=Wed, 15-Jul-2026 05:21:35 GMT; HttpOnly; Path=/; SameSite=Lax
    Server: Werkzeug/1.0.1 Python/3.10.18
    Date: Wed, 08 Jul 2026 05:21:35 GMT
body bytes: 229

========================================================================
CONDITION 3  GET /             (authenticated session)
------------------------------------------------------------------------
request : GET http://127.0.0.1:7777/
status  : HTTP/1.0 302 FOUND
response headers:
    Content-Type: text/html; charset=utf-8
    Content-Length: 229
    Location: http://127.0.0.1:7777/dashboard/
    Vary: Cookie
    Set-Cookie: slapp=<SIGNED_VALUE_REDACTED>; Expires=Wed, 15-Jul-2026 05:21:35 GMT; HttpOnly; Path=/; SameSite=Lax
    Server: Werkzeug/1.0.1 Python/3.10.18
    Date: Wed, 08 Jul 2026 05:21:35 GMT
body bytes: 229

========================================================================
CONDITION 4  GET /dashboard/   (authenticated session)
------------------------------------------------------------------------
request : GET http://127.0.0.1:7777/dashboard/
status  : HTTP/1.0 200 OK
response headers:
    Content-Type: text/html; charset=utf-8
    Content-Length: 781374
    Vary: Cookie
    Set-Cookie: slapp=<SIGNED_VALUE_REDACTED>; Expires=Wed, 15-Jul-2026 05:21:35 GMT; HttpOnly; Path=/; SameSite=Lax
    Server: Werkzeug/1.0.1 Python/3.10.18
    Date: Wed, 08 Jul 2026 05:21:35 GMT
body bytes: 781374

========================================================================
CONDITION 5  GET /             (fresh session, no cookies)
------------------------------------------------------------------------
request : GET http://127.0.0.1:7777/
status  : HTTP/1.0 302 FOUND
response headers:
    Content-Type: text/html; charset=utf-8
    Content-Length: 229
    Location: http://127.0.0.1:7777/auth/login
    Vary: Cookie
    Set-Cookie: slapp=<SIGNED_VALUE_REDACTED>; Expires=Wed, 15-Jul-2026 05:21:35 GMT; HttpOnly; Path=/; SameSite=Lax
    Server: Werkzeug/1.0.1 Python/3.10.18
    Date: Wed, 08 Jul 2026 05:21:35 GMT
body bytes: 229

========================================================================
SESSION COOKIE DECODE  (authenticated 'slapp' cookie, decoded with FLASK_SECRET)
------------------------------------------------------------------------
decoded payload keys: ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
_user_id: fc14048c-8214-4d7a-a559-31c0df0775cd
```

Server-side, the corresponding log lines emitted by the serving child (PID `15181`) during the same
probe run — captured from the server's stdout, unedited, each self-citing its source `file:line`:

```text
2026-07-08 05:21:35,021 - SL - DEBUG - 15181 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.022771835327148438
2026-07-08 05:21:35,273 - SL - DEBUG - 15181 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-07-08 05:21:35,274 - SL - DEBUG - 15181 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 05:21:35,274 - SL - DEBUG - 15181 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2491466999053955
2026-07-08 05:21:35,283 - SL - DEBUG - 15181 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.004232883453369141
2026-07-08 05:21:35,654 - SL - DEBUG - 15181 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.36823105812072754
2026-07-08 05:21:35,662 - SL - DEBUG - 15181 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0007114410400390625
```

The client-side conditions map one-to-one onto these server-side lines (same run, same serving child
PID `15181`): CONDITION 1 → the `GET /auth/login 200`; CONDITION 2 → the two `after_login()` lines
followed by `POST /auth/login 302`; CONDITION 3 → `GET / 302`; CONDITION 4 → `GET /dashboard/ 200`;
CONDITION 5 → the final `GET / 302`.

Three things are directly observable here:

- The `after_login()` line `log user <User 1 John Wick john@wick.com> in`
  (`app/auth/views/login_utils.py:L35`) shows the resolved user object — confirming the password
  check succeeded and Flask-Login's `login_user()` was invoked.
- The subsequent authenticated `GET /dashboard/` returns `200`, while an unauthenticated `GET /`
  returns `302` to `/auth/login` — the two branches of the identity flow (see C5).
- The decoded `slapp` cookie's `_user_id` is the `alternative_id` UUID
  `fc14048c-8214-4d7a-a559-31c0df0775cd` (not the primary key `1`) — the runtime identity token
  analyzed in §2.2 and C4.

## 2.5 How the auth context is carried through the request

Auth context is carried by **the signed `slapp` session cookie plus the two per-request hooks**:

1. On login, `login_user(user)` (`app/auth/views/login_utils.py:L36`) writes
   `_user_id = user.get_id()` — the `alternative_id` UUID (`app/models.py:L597`) — into the signed
   `slapp` cookie (observed `Set-Cookie: slapp=<SIGNED_VALUE_REDACTED>; Expires=...; HttpOnly; Path=/; SameSite=Lax`, §1.5).
2. On each subsequent request, `before_request` (`server.py:L257-L270`) seeds `g.start_time`; then
   Flask-Login reads the cookie and calls `load_user(alternative_id)` (`server.py:L220-L222`), which
   resolves `current_user` from `User.get_by(alternative_id=...)`.
3. The view executes with `current_user` populated; `after_request` (`server.py:L272-L299`) then logs
   the request via `LOG.d(...)` at `server.py:L284`.

If no valid `slapp` cookie is present, `load_user` is not able to resolve a user and `current_user`
is the Flask-Login anonymous user — which is why a fresh `GET /` redirects to `/auth/login`
(observed in CONDITION 5 above).

## 2.6 Identity-flow diagram

```mermaid
flowchart TD
    A[HTTP request arrives] --> B[ProxyFix wraps WSGI app<br/>server.py:L142]
    B --> C[before_request hook sets g.start_time<br/>server.py:L257-L270]
    C --> D{signed slapp session cookie present?}
    D -- yes --> E[Flask-Login user_loader: load_user alternative_id<br/>server.py:L220-L222]
    E --> F[User.get_by alternative_id yields current_user<br/>get_id returns alternative_id, app/models.py:L595-L597]
    D -- no --> G[current_user = Anonymous]
    F --> H[View / blueprint handles request]
    G --> H
    H --> I[after_request logs line via LOG.d<br/>server.py:L284]
    I --> J[Response returned]
```

---

# Section 3 — Background work

## 3.1 Does the web app auto-start any jobs or schedulers? No — observed negative result

Starting `server.py` starts **no** background jobs, schedulers, threads, or subprocesses of its own.
Beyond handling incoming HTTP requests (and the reloader supervisor described in C1), nothing else
runs.

This was verified by grepping the web entry point for every spawning / scheduling primitive and for
the names of the background modules:

Command:

```bash
grep -nE 'Thread|threading|subprocess|Process|multiprocessing|scheduler|apscheduler|job_runner|event_listener|monitoring|email_handler|cron' server.py
echo "EXIT_CODE=$?"
```

Observed output:

```text
EXIT_CODE=1
```

The grep produced **zero matching lines** and exited `1` (grep's "no match" status). This was
confirmed identically against both the working-tree copy and the running container's
`/app/server.py`. So `server.py` does not spawn threads/processes and does not even *import* the
background modules — corroborating the runtime observation that no worker banners or scheduler logs
appear on startup (§1.4) and that the only recurring runtime log line is the per-request
`after_request` entry (§2.1).

## 3.2 Where background work actually lives — independent process entry points

Background responsibilities are carried by **separate standalone scripts**, each launched as its own
process (each has its own `if __name__ == "__main__":` guard). None of them is started by the web
app:

| Script | Role | Entry-point evidence |
|--------|------|----------------------|
| `job_runner.py` | Drains Redis-backed jobs | `def process_job(job: Job):` (`job_runner.py:L188`); `if __name__ == "__main__":` (`job_runner.py:L329`); `while True:` loop (`job_runner.py:L330`) |
| `cron.py` | Scheduled maintenance tasks (invoked by yacron) | `import argparse` (`cron.py:L1`); `if __name__ == "__main__":` (`cron.py:L1262`); `parser = argparse.ArgumentParser()` (`cron.py:L1264`) |
| `event_listener.py` | PostgreSQL `LISTEN/NOTIFY` event processing | `def main(...)` (`event_listener.py:L29`); `runner.run()` (`event_listener.py:L47`); `if __name__ == "__main__":` (`event_listener.py:L94`) |
| `monitoring.py` | Metrics-collection daemon | `if __name__ == "__main__":` (`monitoring.py:L157`); `while True:` loop (`monitoring.py:L159`) |
| `email_handler.py` | Inbound SMTP (aiosmtpd) | `import argparse` (`email_handler.py:L33`); `if __name__ == "__main__":` (`email_handler.py:L2396`) |

The scheduled jobs themselves are declared for **yacron** in `crontab.yml` (present, ~2.8 KB) and
`crontab-all-hosts.yml` (present, ~0.2 KB). Every schedule in `crontab.yml` invokes the standalone
cron script — the observed command template is `python /code/cron.py -j <job>` — for jobs such as
`stats`, `delete_old_monitoring`, `check_custom_domain`, `check_hibp`, `notify_hibp`, `delete_logs`,
`delete_old_data`, `poll_apple_subscription`, `notify_trial_end`, `notify_manual_subscription_end`,
`notify_premium_end`, `delete_scheduled_users`, `send_undelivered_mails`, `clear_alias_audit_log`,
and `clear_user_audit_log`. These run as separate `cron.py` processes under yacron — **not** inside
`server.py`.

## 3.3 Contrast with the production web tier

> **_Inferred_ (read from source, not executed).** The production tier was **not** run — running
> Gunicorn is outside this investigation's scope, which targets the dev server's observed output. The
> statements in this subsection are read from the `Dockerfile` and `wsgi.py`, not observed at runtime.

In production the same application object is served by Gunicorn rather than the dev server. The
`Dockerfile` declares:

- `EXPOSE 7777` — `Dockerfile:L44`.
- `CMD ["gunicorn", "wsgi:app", "-b", "0.0.0.0:7777", "-w", "2", "--timeout", "15"]` — `Dockerfile:L47`
  (effective command: `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`).

`wsgi:app` is the same factory output — `from server import create_app` (`wsgi.py:L1`) and
`app = create_app()` (`wsgi.py:L3`). So production and dev bind the **same port 7777** and the
**same application**; the differences are the server (Gunicorn vs. the Werkzeug dev server), the
binding host (`0.0.0.0` vs. `127.0.0.1`), worker count, and the absence of the debug reloader.
Neither tier auto-starts the background workers of §3.2. _(The dev-side facts here — port 7777 and
`create_app()` — are observed in §1.4/§1.5; the Gunicorn-side facts are inferred from the
`Dockerfile` as noted above.)_

---

# Critical runtime nuances (where observed behavior differs from a naive code reading)

These five points are the cases where simply reading the code would mislead you; each is grounded in
the observed output above.

## C1 — The startup output prints twice (Werkzeug reloader parent/child)

Because the dev server runs with `app.run(debug=True, ...)` (`server.py:L588`), Werkzeug's
auto-reloader is active. The reloader runs the program in **two processes**: a supervisor (parent)
that watches files, and a child that actually serves requests. Consequently the configuration
`print(...)` statements and the `>>> init logging <<<` marker execute **once per process**, which is
why §1.4 shows every config line twice.

This is confirmed by the two PIDs in the captured output and by inspecting each process's
environment:

```text
PID 15160  PPID 1       WERKZEUG_RUN_MAIN unset   -> reloader supervisor (printed config #1 + the Flask banner, then spawned the child)
PID 15181  PPID 15160   WERKZEUG_RUN_MAIN=true    -> serving worker      (printed config #2; actually handles requests)
```

The Flask banner (`* Serving Flask app ...` … `* Debug mode: on`) appears **only once** (in the
supervisor block); the serving child's block ends at the `load words file` line and has no banner.
All per-request logs in §2.4 were emitted by the serving child (PID 15181).

This is the framework's documented reloader behavior, not a SimpleLogin quirk. The official Werkzeug
documentation for `run_simple()` describes the `use_reloader` parameter as: *"Use a reloader process
to restart the server process when files are changed."*
([werkzeug.palletsprojects.com/en/stable/serving](https://werkzeug.palletsprojects.com/en/stable/serving/)).
That "reloader process" is the observed supervisor (PID 15160) and the "server process" is the
observed child (PID 15181); because `server.py` is executed top-to-bottom in *each* process, the
`app/config.py` `print(...)` lines and the `>>> init logging <<<` marker run once per process — which
is why §1.4 shows every config line twice. The same mechanism is present in the pinned Werkzeug 1.0.1:
the reloader marks its serving subprocess with the environment variable `WERKZEUG_RUN_MAIN=true`,
which is exactly what the captured environment above shows on the serving child (PID 15181) and not on
the supervisor (PID 15160). The reloader can be disabled with `use_reloader=False`, which collapses
the two processes into one and prints the config block only once.

## C2 — Werkzeug's `* Running on ...` line and per-request access logs are ABSENT

A naive reader expects the familiar `* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)`
line and Werkzeug's default `"GET / HTTP/1.1" 200 -` access logs. **Neither appears** in the
observed output. The reason is that the app disables the Werkzeug logger:
`logging.getLogger("werkzeug")` (`app/log.py:L70`) → `.disabled = True` (`app/log.py:L71`).

Therefore the empirical readiness signal is **not** "Running on"; it is the Flask banner ending in
`* Debug mode: on` plus the `>>> URL:` (`app/config.py:L80`) and `>>> init logging <<<`
(`app/log.py:L67`) markers, with actual serving readiness confirmed by the `curl` handshake in §1.5.
The only per-request log that appears is SimpleLogin's own `after_request` line (`server.py:L284`),
seen in §2.4.

## C3 — The banner says `Environment: production` even though `Debug mode: on`

The captured banner reads:

```text
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
 * Debug mode: on
```

This is the genuine, unedited output of Flask 1.1.2 when `FLASK_ENV` is unset: Flask defaults the
reported environment to `production` while `debug=True` (from `server.py:L588`) still enables debug
mode and the reloader. It is reported here **as-is** and is not "corrected"; it is simply how Flask
1.1.2 prints under the canonical dev configuration (no `FLASK_ENV` in the canonical `.env`).

## C4 — Runtime identity is keyed on `alternative_id` (a UUID), not the primary key

As proven in §2.2: `User.get_id()` returns `alternative_id` (`app/models.py:L595-L597`); the signed
`slapp` session cookie's `_user_id` was observed to equal the `alternative_id` UUID
(`fc14048c-8214-4d7a-a559-31c0df0775cd`), **not** the primary key `1`; and `load_user`
(`server.py:L220-L222`) reverses that UUID back into the `User`. This is the concrete answer to "how
the user identity is determined at runtime."

## C5 — Both conditions exercised: unauthenticated vs. authenticated

The identity branch was exercised on **both** paths, not just the happy path:

```text
# Unauthenticated (fresh session)
GET /            -> 302  Location: http://127.0.0.1:7777/auth/login

# Authenticated (after login as john@wick.com)
GET /dashboard/  -> 200
```

The unauthenticated `GET /` → `302 /auth/login` corresponds to the `current_user = Anonymous` branch
of the §2.6 diagram; the authenticated `GET /dashboard/` → `200` corresponds to the resolved
`current_user` branch. Both were observed (client-side in §2.4 CONDITION 4 & CONDITION 5, and
server-side in the §2.4 log block).

The branch is not merely observed — it is the literal `if/else` in the root `index` view:
`if current_user.is_authenticated:` (`server.py:L252`) redirects to `dashboard.index`
(`server.py:L253`), `else:` (`server.py:L254`) redirects to `auth.login` (`server.py:L255`). The two
observed `GET /` outcomes map one-to-one onto these two lines, so the identity resolution described
in §2.2 is what selects the branch at runtime.

---

# Read-only scope verification

The investigation was strictly read-only with respect to the SimpleLogin codebase. All temporary
probe scripts and captured logs were kept **outside** the repository tree (e.g. `/tmp/probe_auth.py`,
`/tmp/server.log`) and removed afterward. The only file added to the repository is this answer
document.

Verification was performed after authoring this document and before committing it. Command:

```bash
git status --porcelain
```

Observed — the default porcelain form **collapses** the entirely-untracked `blitzy/` directory to a
single entry:

```text
?? blitzy/
```

To confirm the exact per-file content of that new tree, the untracked-files-all form was used.
Command:

```bash
git status --porcelain --untracked-files=all
```

Observed (only the new documentation file is present; **no** SimpleLogin source file is modified,
added, or deleted):

```text
?? blitzy/documentation/app_2cd6ee777f8c.md
```

(The sibling directories `blitzy/screen_recordings/` and `blitzy/screenshots/` are empty and are not
shown because git does not track empty directories.) After this document is committed,
`git status --porcelain` reports a clean tree.

---

# Summary — direct answers to each named item

**Startup in dev mode.** `python server.py` runs the module and calls
`app.run(debug=True, port=7777)` (`server.py:L588`), building the app via `create_app()`
(`server.py:L139`). Configuration is loaded by `python-dotenv` from `.env` in `app/config.py`
(`L9`, `L65`, `L71`), which prints `>>> URL: http://localhost:7777` (`L80`) and four more config
lines (`L123`, `L217`, `L262`, `L328`). Logging init prints `>>> init logging <<<` (`app/log.py:L67`)
and disables the Werkzeug logger (`app/log.py:L70-L71`). The readiness signal is the Flask banner
ending `* Debug mode: on` (the `* Running on ...` line is absent — C2), and the server binds
`127.0.0.1:7777` (confirmed by `curl`, §1.5). The startup prints twice due to the reloader (C1).

**Authenticated request handling.** The request enters the `ProxyFix`-wrapped WSGI app
(`server.py:L142`) built by `create_app()`; `before_request`/`after_request` hooks bracket it
(`server.py:L257`, `L272`, log at `L284`). Identity is resolved by Flask-Login's `load_user`
(`server.py:L220-L222`) from the signed `slapp` cookie, whose `_user_id` is the user's
`alternative_id` UUID (`app/models.py:L595-L597`) — observed to be
`fc14048c-8214-4d7a-a559-31c0df0775cd` for `john@wick.com`, not the primary key (C4). Login was
driven through the real `/auth/login` view (`app/auth/views/login.py:L21-L72`) →
`after_login()` (`app/auth/views/login_utils.py:L12`, `login_user()` at `L36`), observed to log
`log user <User 1 John Wick john@wick.com> in` and redirect to `/dashboard/` (`302`).

**Background work.** `server.py` auto-starts **no** background jobs or schedulers — the spawn/schedule
grep returns zero matches, exit code 1 (§3.1). Background work runs as independent process entry
points — `job_runner.py`, `cron.py`, `event_listener.py`, `monitoring.py`, `email_handler.py` — with
yacron schedules in `crontab.yml`/`crontab-all-hosts.yml` (§3.2). Production serves the same app via
Gunicorn on the same port 7777 (`Dockerfile:L44`, `L47`; `wsgi.py`), also without auto-starting those
workers (§3.3).
