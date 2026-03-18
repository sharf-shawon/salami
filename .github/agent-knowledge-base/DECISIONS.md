# Decisions

A log of architectural, library, and design decisions, the alternatives considered, and the trade-offs made.
Search this file before proposing a change that might conflict with a recorded decision.

---

## Entry Template

Copy and fill in this template for every new entry:

```
### [YYYY-MM-DD] <Short title>

**Module/Area:** <e.g. users, api, auth, storage, ci>
**Decision:** <What was decided>
**Alternatives considered:** <What other options were evaluated>
**Rationale:** <Why this option was chosen>
**Trade-offs:** <Known downsides or accepted limitations>
**Reference:** <File path(s), PR, issue, or external docs>
```

---

## Entries

### [2026-03-17] Use Cookiecutter Django as project scaffold

**Module/Area:** project, architecture
**Decision:** Bootstrap the project using the [Cookiecutter Django](https://github.com/cookiecutter/cookiecutter-django) template.
**Alternatives considered:** Starting from a blank Django project; using a custom internal template.
**Rationale:** Cookiecutter Django provides production-ready defaults for Docker, CI, settings separation, custom User model, Allauth, DRF, S3 storage, Argon2 password hashing, and pre-commit hooks. It encodes years of community best practice and significantly reduces boilerplate setup time.
**Trade-offs:** The scaffold includes some features (e.g. Traefik, Sphinx docs, multiple language locales) that may not all be used immediately, adding structural complexity. Updates to the scaffold must be applied manually.
**Reference:** `README.md`, `pyproject.toml`

---

### [2026-03-17] Custom User model (`salami.users.User`)

**Module/Area:** users, auth
**Decision:** Use a custom User model (`salami.users.User`) as `AUTH_USER_MODEL` from the start of the project.
**Alternatives considered:** Using Django's built-in `auth.User` and extending it via a `Profile` model.
**Rationale:** Django's documentation strongly recommends defining a custom User model at project start to allow future flexibility (e.g. adding fields, changing username constraints). Retrofitting a custom User model after initial migrations is extremely disruptive.
**Trade-offs:** Slightly more initial setup; all foreign keys to the User model must use `settings.AUTH_USER_MODEL` instead of `'auth.User'`.
**Reference:** `salami/users/models.py`, `config/settings/base.py`

---

### [2026-03-17] Argon2 as primary password hasher

**Module/Area:** auth, security
**Decision:** Use `Argon2PasswordHasher` as the first (primary) entry in `PASSWORD_HASHERS`.
**Alternatives considered:** Keeping Django's default PBKDF2 hasher; using bcrypt.
**Rationale:** Argon2 is the winner of the Password Hashing Competition and is recommended for new projects. It provides memory-hard hashing, making brute-force attacks significantly more expensive than PBKDF2.
**Trade-offs:** Requires the `argon2-cffi` package (already in dependencies). Passwords hashed with older algorithms will be automatically re-hashed on next login.
**Reference:** `config/settings/base.py` (`PASSWORD_HASHERS`), `pyproject.toml` (`argon2-cffi`)

---

### [2026-03-17] `uv` as the Python package manager

**Module/Area:** tooling, dependencies
**Decision:** Use `uv` (Astral) for Python dependency management and virtual environment handling, replacing pip/pip-tools/poetry.
**Alternatives considered:** pip + pip-tools; Poetry; PDM.
**Rationale:** `uv` is significantly faster than pip for installs and lock-file resolution, has a compatible `pyproject.toml`-based workflow, and produces deterministic installs via `uv.lock`. It integrates well with the existing `pyproject.toml` structure.
**Trade-offs:** `uv` is newer and less universally known than pip or Poetry. The `uv.lock` format is uv-specific and not interoperable with other tools. CI must have `uv` available.
**Reference:** `pyproject.toml`, `uv.lock`, `compose/local/django/Dockerfile`

---

### [2026-03-17] Ruff as the sole linter and formatter (replacing flake8/isort/black)

**Module/Area:** tooling, style
**Decision:** Use `ruff` for both linting and formatting. No separate `flake8`, `isort`, or `black` invocations.
**Alternatives considered:** flake8 + isort + black (the traditional trio); pylint.
**Rationale:** Ruff is orders of magnitude faster than the traditional trio, implements nearly all of their rules, and consolidates configuration in `pyproject.toml`. A single tool is simpler to maintain in CI and pre-commit.
**Trade-offs:** Ruff's formatter output is slightly different from Black's in edge cases. The extensive rule set (`pyproject.toml` selects ~25 rule groups) means new code must pass a high bar — which is intentional.
**Reference:** `pyproject.toml` (`[tool.ruff]`), `.pre-commit-config.yaml`

---

### [2026-03-17] Django Unfold for the admin interface

**Module/Area:** admin, ui
**Decision:** Use `django-unfold` as the admin interface theme/framework instead of the default Django admin or `django-jet`.
**Alternatives considered:** Default Django admin; django-jet; django-grappelli; jazzmin.
**Rationale:** Unfold provides a modern, clean admin UI with Bootstrap 5 integration, supports Django 6, and is actively maintained. It requires minimal configuration and produces a significantly better user experience than the default admin.
**Trade-offs:** Unfold must be listed before `django.contrib.admin` in `INSTALLED_APPS`. Customisation hooks differ from the default admin.
**Reference:** `config/settings/base.py` (`INSTALLED_APPS`), `config/unfold.py`

---

### [2026-03-17] DRF + drf-spectacular for REST API and OpenAPI docs

**Module/Area:** api
**Decision:** Use Django REST Framework for the REST API and `drf-spectacular` for auto-generated OpenAPI 3 documentation (Swagger UI / Redoc).
**Alternatives considered:** Django Ninja; raw Django views with manual OpenAPI docs; DRF + drf-yasg.
**Rationale:** DRF is the de facto standard for Django REST APIs with extensive ecosystem support. `drf-spectacular` supersedes `drf-yasg` for OpenAPI 3 generation and integrates cleanly with DRF's serializer introspection.
**Trade-offs:** DRF adds boilerplate compared to Django Ninja's decorator-based approach. OpenAPI docs are only available to admin users by default (see `SERVE_PERMISSIONS` in `SPECTACULAR_SETTINGS`).
**Reference:** `config/settings/base.py` (`REST_FRAMEWORK`, `SPECTACULAR_SETTINGS`), `config/api_router.py`

---

### [2026-03-17] CORS restricted to `/api/.*` paths only

**Module/Area:** api, security
**Decision:** Set `CORS_URLS_REGEX = r"^/api/.*$"` to restrict CORS headers to API endpoints only.
**Alternatives considered:** Enabling CORS globally; disabling CORS entirely.
**Rationale:** Template-rendered pages use session authentication and do not need CORS. Restricting CORS to the API surface minimises the attack surface and prevents inadvertent cross-origin access to admin or template pages.
**Trade-offs:** Any future non-`/api/` endpoint that needs CORS (e.g. a webhook receiver) must have its path added to `CORS_URLS_REGEX` explicitly.
**Reference:** `config/settings/base.py` (`CORS_URLS_REGEX`)

---

### [2026-03-17] `just` as the local task runner

**Module/Area:** tooling, dx
**Decision:** Use `just` (the command runner) as a thin wrapper around Docker Compose commands for common local development tasks.
**Alternatives considered:** Makefile; shell scripts; Taskfile.
**Rationale:** `just` has cleaner syntax than Makefiles, avoids `.PHONY` boilerplate, supports cross-platform usage, and provides a self-documenting `just --list` command. It delegates to Docker Compose, keeping the actual commands in one place.
**Trade-offs:** Developers need `just` installed locally. The `justfile` does not cover all operations — complex one-off commands still use Docker Compose directly. Per the comment in `justfile`, `just` does not yet manage signals reliably for subprocesses.
**Reference:** `justfile`
