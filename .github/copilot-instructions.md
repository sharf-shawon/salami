# GitHub Copilot Agent Instructions — Salami

> **Always read this file in full before making any changes.** It is the authoritative guide for all Copilot agents working in this repository.

---

## 1. Repository Context

**Salami** is a **Digital Eid Salami Portal** — a Django-based web application for managing the Islamic tradition of monetary gift-giving (salami) during Eid celebrations. It provides a structured, digital platform for users to send, receive, and track Eid salami transactions.

**Primary users:** Individuals and families participating in Eid gift-giving traditions, primarily in Bangladesh and the broader Bengali-speaking diaspora.

**Core problems it solves:**

- Digitises and streamlines the traditionally informal process of Eid salami exchange.
- Provides a secure, authenticated platform for monetary gift management.
- Offers a REST API for potential mobile or third-party integrations.

**Tech stack summary:**

| Layer            | Technology                                         |
| ---------------- | -------------------------------------------------- |
| Language         | Python 3.13                                        |
| Web framework    | Django 6.0                                         |
| REST API         | Django REST Framework (DRF) 3.16 + drf-spectacular |
| Authentication   | django-allauth 65 (username + MFA)                 |
| Database         | PostgreSQL 18 (via psycopg3)                       |
| Cache / Sessions | Redis 8                                            |
| Task runner      | `just` (local), Docker Compose                     |
| Package manager  | `uv` (Python), `npm` (Node)                        |
| Frontend         | Webpack 5 + Bootstrap 5 + Sass                     |
| Admin UI         | django-unfold                                      |
| Storage (prod)   | AWS S3 via django-storages                         |
| Reverse proxy    | Traefik (production)                               |
| ASGI server      | Uvicorn                                            |
| WSGI server      | Gunicorn                                           |
| Containerisation | Docker + Docker Compose                            |
| CI/CD            | GitHub Actions                                     |
| Linter/formatter | Ruff                                               |
| Type checker     | mypy                                               |
| Test framework   | pytest + pytest-django                             |

---

## 2. Project Structure

```
salami/                          # Repository root
├── .devcontainer/               # VS Code Dev Container config
├── .envs/
│   └── .local/
│       ├── .django              # Local Django env vars
│       └── .postgres            # Local PostgreSQL env vars
├── .github/
│   ├── agent-knowledge-base/    # Self-improving knowledge base (agents must update)
│   │   ├── DECISIONS.md
│   │   ├── LEARNINGS.md
│   │   └── MISTAKES.md
│   ├── workflows/
│   │   └── ci.yml               # GitHub Actions: lint + pytest via Docker
│   ├── dependabot.yml
│   └── copilot-instructions.md  # ← This file
├── compose/
│   ├── local/                   # Local Docker build contexts (django, node, docs)
│   └── production/              # Production build contexts (django, postgres, traefik, aws)
├── config/
│   ├── settings/
│   │   ├── base.py              # Shared settings (authoritative defaults)
│   │   ├── local.py             # Local dev overrides (DEBUG=True)
│   │   ├── production.py        # Production settings (HTTPS, S3, Redis)
│   │   └── test.py              # Test-only settings
│   ├── api_router.py            # DRF router — register ViewSets here
│   ├── urls.py                  # Root URL configuration
│   ├── asgi.py                  # ASGI entry point (Uvicorn / WebSockets)
│   ├── wsgi.py                  # WSGI entry point (Gunicorn)
│   ├── unfold.py                # Django Unfold admin customisation
│   └── websocket.py             # WebSocket URL routing
├── salami/                      # Main Django application package
│   ├── users/                   # User management app (custom User model)
│   │   ├── models.py            # Custom User model (AUTH_USER_MODEL)
│   │   ├── views.py             # Template-based views
│   │   ├── forms.py             # Signup/profile forms
│   │   ├── admin.py             # User admin registration
│   │   ├── adapters.py          # django-allauth adapters
│   │   ├── urls.py              # User URL patterns
│   │   ├── api/
│   │   │   ├── views.py         # DRF ViewSets for users
│   │   │   └── serializers.py   # DRF serializers
│   │   ├── tests/               # All user-related tests
│   │   │   ├── factories.py     # factory-boy factories
│   │   │   ├── test_models.py
│   │   │   ├── test_views.py
│   │   │   ├── test_forms.py
│   │   │   ├── test_admin.py
│   │   │   ├── test_urls.py
│   │   │   └── api/             # API-specific tests
│   │   └── migrations/
│   ├── contrib/
│   │   └── sites/migrations/    # Custom sites framework migrations
│   ├── static/                  # Source static files (JS, Sass, images, fonts)
│   ├── templates/               # Django HTML templates
│   └── conftest.py              # pytest fixtures shared across tests
├── tests/                       # Project-level (non-app) tests
├── docs/                        # Sphinx documentation source
├── webpack/                     # Webpack config (common, dev, prod)
├── locale/                      # Translation files (en_US, fr_FR, pt_BR)
├── docker-compose.local.yml     # Local Docker Compose stack
├── docker-compose.production.yml
├── docker-compose.docs.yml
├── justfile                     # `just` task runner shortcuts
├── pyproject.toml               # Python project metadata, Ruff, pytest, mypy, djlint
├── package.json                 # Node dependencies and npm scripts
├── merge_production_dotenvs_in_dotenv.py
└── manage.py
```

### Always read these first

Before making any changes, consult the following files in order:

1. **`.github/copilot-instructions.md`** — this file
2. **`README.md`** — project overview and basic commands
3. **`.github/agent-knowledge-base/LEARNINGS.md`** — non-obvious patterns and lessons
4. **`.github/agent-knowledge-base/MISTAKES.md`** — known mistakes and prevention rules
5. **`.github/agent-knowledge-base/DECISIONS.md`** — architectural decisions and trade-offs
6. **`config/settings/base.py`** — canonical Django settings
7. **`pyproject.toml`** — tooling configuration (Ruff, pytest, mypy, djlint)
8. **`config/api_router.py`** and **`config/urls.py`** — URL and API routing
9. The relevant app's `models.py`, `views.py`, and `tests/` before editing that app

---

## 3. Tooling and Commands

### 3.1 Running Tests

**Always use `pytest`, not `manage.py test`, for automated tests.**

```bash
# Run the full test suite (local, requires PostgreSQL + Redis running)
uv run pytest

# Run tests with coverage
uv run coverage run -m pytest
uv run coverage html
# Open htmlcov/index.html to view the report

# Run a specific test file
uv run pytest salami/users/tests/test_models.py

# Run a specific test by name
uv run pytest -k "test_user_get_absolute_url"

# Run tests via Docker Compose (matches CI exactly)
docker compose -f docker-compose.local.yml run --rm django pytest
```

### 3.2 Running Linters and Formatters

**The canonical linting and formatting tool is `ruff`. Always run it before committing.**

```bash
# Lint (check only)
uv run ruff check salami

# Lint with auto-fix
uv run ruff check salami --fix

# Format
uv run ruff format salami

# Type checking
uv run mypy salami

# Django template linting and reformatting
uv run djlint salami/templates --reformat

# Run all pre-commit hooks against all files (recommended before pushing)
pre-commit run --all-files
```

Individual pre-commit hooks available:

- `ruff-check` — Python linting
- `ruff-format` — Python formatting
- `djlint` — Django template linting
- `django-upgrade` — Auto-upgrade Django syntax to 6.0
- `pyproject-fmt` — Format `pyproject.toml`
- `prettier` — JS/CSS/JSON formatting (2-space indent, single quotes)

### 3.3 Database Migrations

```bash
# Check for missing migrations (run this in CI and before committing schema changes)
docker compose -f docker-compose.local.yml run --rm django python manage.py makemigrations --check

# Create new migrations after model changes
docker compose -f docker-compose.local.yml run --rm django python manage.py makemigrations

# Apply migrations
docker compose -f docker-compose.local.yml run --rm django python manage.py migrate

# Without Docker (requires local DB)
uv run python manage.py makemigrations
uv run python manage.py migrate
```

### 3.4 Running the App Locally

**Option A — Docker Compose (recommended, matches CI):**

```bash
# Start all services (Django, PostgreSQL, Redis, Node/Webpack)
just up
# or: docker compose -f docker-compose.local.yml up -d

# Stop all services
just down

# View logs
just logs
just logs django

# Run a manage.py command
just manage createsuperuser
just manage shell_plus

# Build images
just build
```

Access points:

- Django app: `http://localhost:8000`
- Webpack dev server: `http://localhost:3000`

**Option B — Manual (requires local PostgreSQL and Redis):**

```bash
uv sync                          # Install Python dependencies
npm install                      # Install Node dependencies
uv run python manage.py migrate  # Apply migrations
uv run python manage.py createsuperuser

# Start ASGI dev server (with hot reload)
uv run uvicorn config.asgi:application --host 0.0.0.0 --reload --reload-include '*.html'

# Start Webpack dev server (separate terminal)
npm run dev
```

### 3.5 Django System Check

Always run this before submitting changes:

```bash
docker compose -f docker-compose.local.yml run --rm django python manage.py check
```

---

## 4. Coding and Quality Standards

### 4.1 Language and Framework Conventions

- **Python version:** 3.13 (strictly enforced via `.python-version` and `pyproject.toml`).
- **Type hints:** Required on all new functions and methods. Use `from __future__ import annotations` where appropriate for forward references. Run `mypy` to validate.
- **Imports:** Follow Ruff `isort` rules with `force-single-line = true`. One import per line.
- **String quotes:** Use double quotes (`"`) for Python strings (Ruff enforces this).
- **Line length:** 119 characters maximum (enforced by djLint for templates; Ruff for Python).
- **Naming conventions:**
  - `snake_case` for variables, functions, and modules.
  - `PascalCase` for classes.
  - `SCREAMING_SNAKE_CASE` for module-level constants.
  - URL names: `app_name:view_name` pattern (e.g. `users:detail`).
  - API endpoint names follow DRF router conventions.

### 4.2 Model and Migration Patterns

- All new apps must define a `default_app_config` or use `AppConfig` in `apps.py`.
- All new models must be registered in the app's `admin.py`.
- **Custom User model** (`salami.users.User`) is `AUTH_USER_MODEL`. Never reference `auth.User` directly; always use `get_user_model()` or `settings.AUTH_USER_MODEL`.
- Use `TimeStampedModel` from `django-model-utils` for audit fields (`created`, `modified`) on all new models.
- Always set `verbose_name` and `verbose_name_plural` on `Meta` class.
- Every model change **must** produce a corresponding migration. Run `makemigrations --check` to verify.
- **Never delete or rewrite existing migrations.** Create new corrective migrations if needed.
- Use `DEFAULT_AUTO_FIELD = BigAutoField` (project default, inherited from Django's default).
- Migrations for `django.contrib.sites` live in `salami/contrib/sites/migrations/`.

### 4.3 API Design Rules

- All REST API ViewSets are registered in `config/api_router.py`. Do not add API URLs anywhere else.
- API URLs are prefixed with `/api/` (see `config/urls.py`).
- **Always declare explicit `permission_classes`** on every ViewSet and APIView. Do not rely solely on the global default.
- Use DRF's `DefaultRouter` in DEBUG mode and `SimpleRouter` in production (already configured in `api_router.py`).
- **Error responses** must use DRF's standard format:
  ```json
  { "detail": "Error message." }
  ```
  For field errors:
  ```json
  { "field_name": ["Error message."] }
  ```
- **Pagination:** Use DRF's built-in pagination classes. Do not return unbounded list responses.
- **OpenAPI documentation** is auto-generated via `drf-spectacular`. Annotate ViewSets with `@extend_schema` decorators when the auto-generated docs are insufficient.
- API versioning: currently unversioned (v1 implicit). Any new version must be discussed and agreed upon before implementation.
- CORS is restricted to `/api/.*` paths (see `CORS_URLS_REGEX` in `base.py`). Do not expand CORS rules without explicit approval.

### 4.4 Query and Performance Rules

- **Avoid N+1 queries.** Use `select_related()` for ForeignKey/OneToOne and `prefetch_related()` for ManyToMany/reverse FK relationships.
- Use `only()` or `defer()` to limit columns when fetching large datasets and not all fields are needed.
- Database transactions are atomic per request (`ATOMIC_REQUESTS = True`). Do not spawn threads or async tasks that bypass the request transaction without explicit justification.
- Use `QuerySet.exists()` instead of `bool(QuerySet)` for existence checks.
- Use `QuerySet.count()` instead of `len(QuerySet)` for counting.
- Avoid raw SQL unless there is a documented, unavoidable performance reason. If raw SQL is used, it **must** be parameterised — never use string formatting or f-strings to build SQL.

### 4.5 Minimum Expectations per Change Type

**New feature:**

- Unit tests covering the happy path and primary edge cases.
- Integration/API tests if API endpoints are involved.
- Docstring on all new public functions/methods/classes.
- Update or create documentation in `docs/` if the feature is user-facing or architecturally significant.
- Add a `LEARNINGS.md` entry if new patterns were established.
- Add a `DECISIONS.md` entry if a non-trivial architectural or library decision was made.

**Bug fix:**

- A regression test that would have caught the bug (add to the relevant `tests/` directory).
- A `MISTAKES.md` entry describing the bug, root cause, fix, and prevention rule.

**Refactor:**

- No observable behaviour change.
- Existing tests must continue to pass without modification (unless tests themselves were incorrect).
- Test coverage must not decrease.
- Add a `DECISIONS.md` entry if the refactor changes a significant pattern.

---

## 5. Conventional Commits

**All commits in this repository must use the Conventional Commits format.**

### 5.1 Format

```
<type>(<scope>): <short description>

[optional body]

[optional footer(s)]
```

- **type**: one of the allowed types (see below).
- **scope**: optional; use the Django app name, module, or affected area (e.g. `users`, `api`, `migrations`, `ci`, `deps`).
- **short description**: imperative, lowercase, no trailing period, ≤72 characters.
- **body**: explain _why_ and _what_, not _how_. Wrap at 72 characters.
- **footer**: reference issues (`Closes #123`), breaking changes (`BREAKING CHANGE: ...`).

### 5.2 Allowed Types

| Type       | When to use                                                  |
| ---------- | ------------------------------------------------------------ |
| `feat`     | A new user-facing feature                                    |
| `fix`      | A bug fix                                                    |
| `docs`     | Documentation changes only                                   |
| `style`    | Formatting, whitespace — no logic change                     |
| `refactor` | Code restructuring with no behaviour change                  |
| `perf`     | Performance improvement                                      |
| `test`     | Adding or correcting tests                                   |
| `chore`    | Build, tooling, dependency updates                           |
| `ci`       | Changes to GitHub Actions or CI configuration                |
| `security` | Security-related fixes or hardening                          |
| `kb`       | Knowledge-base-only updates (LEARNINGS, MISTAKES, DECISIONS) |

> Use `kb` exclusively for commits that only update files in `.github/agent-knowledge-base/`. This makes knowledge base updates easy to trace in the git history.

### 5.3 Examples

```
feat(users): add MFA enforcement for admin accounts

fix(api): return 403 instead of 500 for unauthenticated webhook calls

docs(users): document custom User model field constraints

refactor(users): extract salami_amount validation to model method

test(users): add regression test for duplicate email signup

chore(deps): bump django from 6.0.2 to 6.0.3

ci: cache Docker layers per PR to reduce build time

security(settings): enforce HSTS and secure cookies in production

kb: record decision to use Argon2 as primary password hasher

perf(api): add select_related for user FK in salami list view

style: apply ruff format to salami/users/models.py
```

---

## 6. Security and Safety Rules

### 6.1 Secrets and Configuration

- **Never commit secrets, credentials, API keys, or tokens to source code or configuration files.**
- All sensitive values must be provided via environment variables, read through `django-environ`'s `env()` calls in `config/settings/`.
- Local secrets belong in `.envs/.local/` files, which are git-ignored.
- Production secrets are injected at deployment time and never stored in the repository.
- The `SECRET_KEY` in `config/settings/local.py` is intentionally insecure and must not be used in any non-local environment.

### 6.2 Database Access

- Use the Django ORM for all database access by default.
- Raw SQL is permitted only when the ORM cannot express the query efficiently and the query is documented with a justification comment.
- **All raw SQL must use parameterised queries** (`cursor.execute(sql, params)`) — never use string formatting, f-strings, or `%` interpolation to construct SQL queries.

### 6.3 Views and API Permissions

- **All DRF ViewSets and APIViews must declare explicit `permission_classes`.**
- The global default (`IsAuthenticated`) is a safety net, not a substitute for per-view declarations.
- Template-based views must use `LoginRequiredMixin` or `@login_required` where authentication is required.
- Never use `permission_classes = []` or `AllowAny` on sensitive endpoints without explicit human approval and a documented justification.

### 6.4 CORS

- CORS is restricted to `^/api/.*$` paths via `CORS_URLS_REGEX`.
- `CORS_ALLOWED_ORIGINS` and `CORS_ALLOW_ALL_ORIGINS` must never be set to blanket-allow all origins in production.
- Any change to CORS configuration requires explicit human approval.

### 6.5 CSRF

- CSRF protection is enabled globally via `CsrfViewMiddleware` (see `MIDDLEWARE` in `base.py`).
- Do not use `@csrf_exempt` without documented justification and explicit human approval.
- DRF API endpoints using `SessionAuthentication` enforce CSRF automatically; do not disable it.

### 6.6 Logging

- **Never log passwords, authentication tokens, session keys, payment data, or any sensitive personal information.**
- Log at `INFO` level for normal operations, `WARNING` for recoverable issues, and `ERROR`/`CRITICAL` for failures.
- Do not add `print()` statements to production code; use the `logging` module.

### 6.7 General Security Posture

- Do not weaken any security setting (`SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `SECURE_SSL_REDIRECT`, `X_FRAME_OPTIONS`, etc.) without explicit human approval.
- Do not bypass Django's authentication or permission system.
- Do not introduce new dependencies with known security vulnerabilities.

---

## 7. Agent Boundaries and Guardrails

### ✅ Always do

- Follow Conventional Commit conventions for every commit.
- Run `pre-commit run --all-files` (or equivalent) before pushing.
- Run the full test suite and confirm it passes before opening a PR.
- Run `python manage.py check` to catch Django configuration errors.
- Read the relevant sections of the knowledge base (`LEARNINGS.md`, `MISTAKES.md`, `DECISIONS.md`) before starting any significant task.
- Add or update knowledge base entries at the end of any task that yields a new learning, decision, or bug fix.
- Declare explicit `permission_classes` on every new view.
- Use the ORM for database access; parameterise any raw SQL.
- Keep type hints current; run `mypy` after changes to Python files.

### ⚠️ Ask first (do not proceed without explicit human confirmation)

- Database schema changes (new models, altering fields, adding indexes).
- Adding or upgrading Python or Node dependencies.
- Changing or extending public API contracts (new endpoints, field changes, removed endpoints).
- Modifying CI/CD workflows (`.github/workflows/`) or production settings (`config/settings/production.py`).
- Large-scale refactors or changes that affect more than one Django app.
- Architectural changes (e.g. introducing a new service layer, task queue, or caching strategy).
- Expanding security-sensitive settings (CORS origins, CSRF exemptions, permission relaxations).
- Any change to `config/settings/production.py`, Traefik configuration, or deployment scripts.

### 🚫 Never do

- Commit secrets, credentials, tokens, or private keys to the repository.
- Delete or rewrite existing database migrations.
- Rewrite git history (force-push, rebase shared branches, amend pushed commits).
- Weaken or bypass security settings, middleware, or permission checks.
- Rename or move major modules, entrypoints, or apps (`salami/users/`, `config/`, `manage.py`) without explicit instruction.
- Use `AllowAny` or empty `permission_classes` on sensitive endpoints.
- Use string formatting or f-strings to construct raw SQL queries.
- Log sensitive personal data, passwords, or tokens.
- Introduce architectural patterns (e.g. CQRS, event sourcing, microservices split) that were not requested.

### Decision rule

> **If you are unsure about models, APIs, file locations, domain rules, security impact, or downstream effects — ask clarifying questions instead of guessing.** Do not invent new high-level architectural patterns unless explicitly requested.

---

## 8. Self-Improving Knowledge Base

The knowledge base lives in `.github/agent-knowledge-base/` and consists of three files:

| File           | Purpose                                                        |
| -------------- | -------------------------------------------------------------- |
| `LEARNINGS.md` | Non-obvious insights, patterns, and lessons that proved useful |
| `MISTAKES.md`  | Mistakes, root causes, fixes, and prevention rules             |
| `DECISIONS.md` | Architectural, library, and design decisions with rationale    |

### 8.1 When to read the knowledge base

- **Before starting any significant task:** search for relevant entries by feature name, module (e.g. `users`, `api`, `migrations`), or domain term (e.g. `MFA`, `salami`, `permissions`).
- Pay particular attention to `MISTAKES.md` prevention rules before touching areas that have had prior issues.

### 8.2 When to update the knowledge base

- **After any significant change** that yields a new insight, pattern, or decision → add to `LEARNINGS.md` or `DECISIONS.md`.
- **After fixing any non-trivial bug** → add to `MISTAKES.md` immediately, including a clear "Prevention rule".
- Use commit type `kb` for commits that only update knowledge base files.

### 8.3 Entry templates

Follow the templates defined at the top of each knowledge base file exactly. This ensures entries are consistent and machine-searchable.

### 8.4 Goal

Build a concise, structured, and continuously updated knowledge base that gives any agent — or new human contributor — a strong, evolving understanding of the project's quirks, decisions, and failure modes.

---

## 9. Documentation Expectations

- Significant new features, architectural changes, or complex subsystems must have a corresponding document in `docs/`.
- Update existing documentation when behaviour changes.
- Each documentation file should cover:
  1. **What** the feature or area does (purpose and user-facing behaviour).
  2. **Key models, services, and API endpoints** involved.
  3. **Important edge cases and constraints** (e.g. permission rules, validation logic, async considerations).
  4. **How to run or test** the relevant functionality (commands, fixtures, expected output).
- Documentation is written in reStructuredText (`.rst`) format to be compatible with Sphinx (see `docs/conf.py`).
- Run `docker compose -f docker-compose.docs.yml up` to preview the documentation site locally.

---

## 10. Post-Implementation Checklist

After completing any task, work through this checklist before committing:

- [ ] **Tests pass:** `docker compose -f docker-compose.local.yml run --rm django pytest` (or `uv run pytest` locally).
- [ ] **Linting and formatting pass:** `pre-commit run --all-files` (or `uv run ruff check salami && uv run ruff format salami`).
- [ ] **Type checking passes:** `uv run mypy salami`.
- [ ] **Django system check passes:** `docker compose -f docker-compose.local.yml run --rm django python manage.py check`.
- [ ] **Migration check passes:** `docker compose -f docker-compose.local.yml run --rm django python manage.py makemigrations --check`.
- [ ] **Knowledge base updated** (if new learnings, decisions, or mistakes arose) with a `kb` commit if knowledge-base-only.
- [ ] **Documentation updated or created** in `docs/` if the change is user-facing or architecturally significant.
- [ ] **Commit message** follows Conventional Commits format with the correct type and scope.
- [ ] **No secrets** introduced into tracked files.
- [ ] **Explicit `permission_classes`** declared on all new views.
