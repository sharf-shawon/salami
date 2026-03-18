# Learnings

Non-obvious insights, patterns, and lessons that proved useful in this codebase.
Search this file by module name, feature, or domain term before starting any significant task.

---

## Entry Template

Copy and fill in this template for every new entry:

```
### [YYYY-MM-DD] <Short title>

**Module/Area:** <e.g. users, api, migrations, settings>
**Context:** <Brief description of the situation or task>
**Learning:** <What was discovered or confirmed>
**Why non-obvious:** <Why this might trip up a developer who hasn't seen it>
**Reference:** <File path(s), PR, or commit if applicable>
```

---

## Entries

### [2026-03-17] Custom User model must always be referenced via `get_user_model()`

**Module/Area:** users, models
**Context:** Project uses a custom `User` model (`salami.users.User`) set as `AUTH_USER_MODEL`.
**Learning:** Never import `django.contrib.auth.models.User` directly. Always use `get_user_model()` at runtime or `settings.AUTH_USER_MODEL` in `ForeignKey` declarations to avoid app-registry conflicts and circular import issues during migration operations.
**Why non-obvious:** Direct imports of `auth.User` appear to work in many situations but fail silently during migrations or when the custom model is swapped.
**Reference:** `salami/users/models.py`, `config/settings/base.py` (`AUTH_USER_MODEL = "users.User"`)

---

### [2026-03-17] `sites` framework migrations live in `salami/contrib/sites/migrations/`

**Module/Area:** migrations, contrib
**Context:** The project overrides the default migration module for `django.contrib.sites`.
**Learning:** Migrations for `django.contrib.sites` are stored in `salami/contrib/sites/migrations/`, not in the default Django location. This is configured via `MIGRATION_MODULES = {"sites": "salami.contrib.sites.migrations"}` in `base.py`.
**Why non-obvious:** Running `makemigrations` for sites-related changes will write to this custom path, not to Django's built-in directory.
**Reference:** `config/settings/base.py`, `salami/contrib/sites/migrations/`

---

### [2026-03-17] `DefaultRouter` vs `SimpleRouter` is environment-dependent

**Module/Area:** api, api_router
**Context:** `config/api_router.py` switches between `DefaultRouter` (DEBUG) and `SimpleRouter` (production).
**Learning:** `DefaultRouter` adds a browsable API root at `/api/` — useful for development. `SimpleRouter` removes it for a cleaner production surface. This is intentional and must be preserved.
**Why non-obvious:** Tests running against `config.settings.test` (which inherits from `local.py` with `DEBUG=True`) will see the API root endpoint; production will not. Test assertions about URL availability should account for this.
**Reference:** `config/api_router.py`, `config/settings/local.py`, `config/settings/test.py`

---

### [2026-03-17] `ATOMIC_REQUESTS = True` wraps every HTTP request in a transaction

**Module/Area:** settings, database
**Context:** `DATABASES["default"]["ATOMIC_REQUESTS"] = True` is set in `base.py`.
**Learning:** Every HTTP request is wrapped in a database transaction. This means: (a) any unhandled exception rolls back all DB changes in that request; (b) long-running operations within a request hold a DB transaction open; (c) background threads or async tasks spawned inside a view do not share the request's transaction.
**Why non-obvious:** Side effects that appear in logs but not in the database after an error are almost always due to this setting rolling back the transaction.
**Reference:** `config/settings/base.py`

---

### [2026-03-17] Webpack stats file must exist before Django starts

**Module/Area:** frontend, webpack
**Context:** `django-webpack-loader` reads `webpack-stats.json` from the project root.
**Learning:** If `webpack-stats.json` does not exist, Django will raise an error on the first request that loads a Webpack bundle. In Docker, the `node` service generates this file. When running Django locally without Docker, run `npm run dev` first (or `npm run build` for a one-off build) to generate the stats file.
**Why non-obvious:** The error is a generic file-not-found error that does not clearly indicate the Webpack stats file is missing.
**Reference:** `config/settings/base.py` (`WEBPACK_LOADER`), `webpack/`, `package.json`

---

### [2026-03-17] Pre-commit hooks include `django-upgrade` targeting Django 6.0

**Module/Area:** tooling, pre-commit
**Context:** `.pre-commit-config.yaml` runs `django-upgrade --target-version 6.0` on all Python files.
**Learning:** Code committed with older Django patterns (e.g. deprecated `ugettext`, old `on_delete` defaults) will be automatically upgraded or will fail the pre-commit check. This is intentional to keep the codebase modern.
**Why non-obvious:** Developers accustomed to older Django versions may be surprised when their code is silently rewritten during the commit hook.
**Reference:** `.pre-commit-config.yaml`

---

### [2026-03-17] Ruff uses `force-single-line = true` for isort

**Module/Area:** tooling, style
**Context:** `pyproject.toml` sets `lint.isort.force-single-line = true`.
**Learning:** Every import must be on its own line. Grouped imports like `from django.db import models, transaction` are not permitted — they must be split into separate lines.
**Why non-obvious:** Standard `isort` defaults allow grouped imports. The Ruff check will fail in CI if grouped imports are present.
**Reference:** `pyproject.toml` (`lint.isort.force-single-line`)
