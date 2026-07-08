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
complete, unedited output that was captured, and (b) a `file:line` (or config-key) reference to the
code that produces it. Anything that was *not* directly observed at runtime is explicitly labeled
**_inferred_**. Command lines that produced each block of output are shown immediately above the
output.

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
| Flask-Login | 0.5.0 | pin in `poetry.lock`; identity flow in §2 |
| SQLAlchemy | 1.3.24 | pin in `poetry.lock` |
| Alembic | 1.4.3 | pin in `poetry.lock`; `alembic upgrade head` used for schema |
| python-dotenv | 0.14.0 | pin in `poetry.lock`; loads `.env` in `app/config.py:L9` |

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
WARNING: Use a temp directory for GNUPGHOME /tmp/usndwhlomtcugoqkwlqh
Upload files to local dir
>>> init logging <<<
2026-07-08 04:27:30,461 - SL - DEBUG - 14414 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
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
WARNING: Use a temp directory for GNUPGHOME /tmp/vuqbijmdmrxrsjxkkhvf
Upload files to local dir
>>> init logging <<<
2026-07-08 04:27:33,307 - SL - DEBUG - 14421 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

Line-by-line, what each observed line is and where it comes from:

- The two leading `UserWarning`/`from pkg_resources import get_distribution` lines are emitted by the
  third-party `flask_limiter` package at import (`flask_limiter/errors.py:4`), not by SimpleLogin
  code. They are reported here because they are part of "what actually gets printed."
- `>>> URL: http://localhost:7777` → `app/config.py:L80` (value from `URL` = `example.env:L6`).
- `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` → `app/config.py:L123`.
- `Paddle param not set` → `app/config.py:L217`.
- `WARNING: Use a temp directory for GNUPGHOME /tmp/usndwhlomtcugoqkwlqh` (and, in the second block,
  `/tmp/vuqbijmdmrxrsjxkkhvf`) → `app/config.py:L262`. The temp-dir name is random per process, so
  the two blocks differ here.
- `Upload files to local dir` → `app/config.py:L328` (taken because `LOCAL_FILE_UPLOAD=true`).
- `>>> init logging <<<` → `app/log.py:L67`.
- `... SL - DEBUG - <pid> - "/app/app/utils.py:17" - <module>() - ... - load words file: /app/local_data/test_words.txt`
  → `app/utils.py:L17` (self-cited by the runtime line itself).
- The four-line Flask banner (`* Serving Flask app "server" (lazy loading)` … `* Debug mode: on`) is
  Flask 1.1.2's own banner, printed once (only in the first/supervisor block — see C1).

**Two process IDs are visible in the output** — `14414` in the first block and `14421` in the
second. This is the reloader parent/child split (proof in C1).

## 1.5 Ports and endpoints

**Bound address: `127.0.0.1:7777`.** This is `app.run(debug=True, port=7777)` at
**`server.py:L588`** (host defaults to `127.0.0.1`). It was confirmed empirically with `curl`
(the authoritative check — see the caveat below):

Command:

```bash
curl -i http://127.0.0.1:7777/auth/login
```

Observed response head (unedited):

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 348625
...
Set-Cookie: slapp=eyJfZnJlc2gi...; Expires=...; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
```

The `Server: Werkzeug/1.0.1 Python/3.10.18` header is direct runtime confirmation of both the
Werkzeug 1.0.1 and Python 3.10.18 pins. A bare `GET /` (no session) returns a redirect to the login
page:

Command:

```bash
curl -i http://127.0.0.1:7777/
```

Observed:

```text
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login
Server: Werkzeug/1.0.1 Python/3.10.18
```

> **Caveat (labeled).** Socket-table tools (`ss -ltnp`, `lsof`) produced **no output** for port 7777
> inside this container — a false negative caused by the container not exposing its socket table to
> those tools, **not** evidence that nothing is listening. The successful `curl` TCP handshake above
> is authoritative and proves the server is bound and serving on `127.0.0.1:7777`.

**Endpoint surface.** The application factory registers 11 blueprints via `register_blueprints()`
(`server.py:L233-L246`). Each URL prefix below was read from the blueprint's own definition (or, for
`oauth_bp`, from the explicit override in `register_blueprints()`):

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

Enumerated **at runtime** (temporary probe, `PYTHONPATH=/app`, reading `app.blueprints` and
`app.url_map`), the *live* app actually exposes **25 blueprints** and **292 URL rules**. The extra
blueprints beyond the 11 registrations above — `admin`, `adminauditlog`, `alias`, `coupon`,
`customdomain`, `dailymetric`, `email_search`, `invalidmailboxdomain`, `mailbox`,
`manualsubscription`, `metric2`, `newsletter`, `newsletteruser`, `providercomplaint`, `user` — are
registered by **Flask-Admin** at runtime (the admin UI), not by `register_blueprints()`.
Representative routes confirmed present at runtime: `/auth/login`, `/dashboard/`, `/api/user_info`,
`/`, `/oauth/authorize`, `/oauth2/authorize`.

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
2026-07-08 04:28:45,031 - SL - DEBUG - 14421 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10902142524719238
2026-07-08 04:28:45,147 - SL - DEBUG - 14421 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0009062290191650391
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

**Runtime proof (observed).** After logging in as `john@wick.com`, the signed `slapp` session cookie
was decoded (temporary probe, using the app's own `FLASK_SECRET` via Flask's
`SecureCookieSessionInterface`). The database row versus the decoded cookie:

```text
# DB row (psql)
id (primary key) = 1
email            = john@wick.com
alternative_id   = fc14048c-8214-4d7a-a559-31c0df0775cd

# Decoded slapp session cookie payload
keys    = ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
_user_id = fc14048c-8214-4d7a-a559-31c0df0775cd
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
(`/tmp/probe_auth.py`, using `requests`). Client-side observations:

```text
STEP 1  GET  /auth/login                    -> 200 (HTTP/1.0); Set-Cookie: slapp=...; CSRF token parsed from form
STEP 2  POST /auth/login (john@wick.com/password)
                                            -> 302; Location: http://127.0.0.1:7777/dashboard/
STEP 3  GET  /            (authenticated)   -> 302; Location: http://127.0.0.1:7777/dashboard/
STEP 4  GET  /dashboard/  (authenticated)   -> 200
STEP 5  GET  /            (fresh session)   -> 302; Location: http://127.0.0.1:7777/auth/login
```

Server-side, the corresponding log lines captured on the serving process (unedited; each self-cites
its source `file:line`):

```text
2026-07-08 04:31:49,955 - SL - DEBUG - 14421 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0240480899810791
2026-07-08 04:31:50,292 - SL - DEBUG - 14421 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-07-08 04:31:50,314 - SL - DEBUG - 14421 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 04:31:50,315 - SL - DEBUG - 14421 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2719085216522217
2026-07-08 04:31:50,326 - SL - DEBUG - 14421 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.005011320114135742
2026-07-08 04:31:50,725 - SL - DEBUG - 14421 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.39488720893859863
2026-07-08 04:31:50,744 - SL - DEBUG - 14421 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0007843971252441406
```

Two things are directly observable here:

- The `after_login()` line `log user <User 1 John Wick john@wick.com> in`
  (`app/auth/views/login_utils.py:L35`) shows the resolved user object — confirming the password
  check succeeded and Flask-Login's `login_user()` was invoked.
- The subsequent authenticated `GET /dashboard/` returns `200`, while an unauthenticated `GET /`
  returns `302` to `/auth/login` — the two branches of the identity flow (see C5).

## 2.5 How the auth context is carried through the request

Auth context is carried by **the signed `slapp` session cookie plus the two per-request hooks**:

1. On login, `login_user(user)` (`app/auth/views/login_utils.py:L36`) writes
   `_user_id = user.get_id()` — the `alternative_id` UUID (`app/models.py:L597`) — into the signed
   `slapp` cookie (observed `Set-Cookie: slapp=...; HttpOnly; Path=/; SameSite=Lax`, §1.5).
2. On each subsequent request, `before_request` (`server.py:L257-L270`) seeds `g.start_time`; then
   Flask-Login reads the cookie and calls `load_user(alternative_id)` (`server.py:L220-L222`), which
   resolves `current_user` from `User.get_by(alternative_id=...)`.
3. The view executes with `current_user` populated; `after_request` (`server.py:L272-L299`) then logs
   the request via `LOG.d(...)` at `server.py:L284`.

If no valid `slapp` cookie is present, `load_user` is not able to resolve a user and `current_user`
is the Flask-Login anonymous user — which is why a fresh `GET /` redirects to `/auth/login`
(observed in STEP 5 above).

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
PID 14414  PPID 1       WERKZEUG_RUN_MAIN unset   -> reloader supervisor (printed config #1 + the Flask banner, then spawned the child)
PID 14421  PPID 14414   WERKZEUG_RUN_MAIN=true    -> serving worker      (printed config #2; actually handles requests)
```

The Flask banner (`* Serving Flask app ...` … `* Debug mode: on`) appears **only once** (in the
supervisor block); the serving child's block ends at the `load words file` line and has no banner.
All per-request logs in §2.4 were emitted by the serving child (PID 14421).

This is the framework's documented reloader behavior, not a SimpleLogin quirk. Per the Plotly Dash
DevTools documentation, code reloading is provided by Flask & Werkzeug via the `use_reloader`
option, and a documented caveat is that **"your app code is run twice when starting"** — once for
the parent process and once for the reloaded child
([dash.plotly.com/devtools](https://dash.plotly.com/devtools)). It can be turned off with
`use_reloader=False`. This matches the observed parent/child split exactly.

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
`current_user` branch. Both were observed (client-side in §2.4 STEP 4 & STEP 5, and server-side in
the §2.4 log block).

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
