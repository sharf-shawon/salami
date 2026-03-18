# Mistakes

A log of mistakes, root causes, fixes, and concrete prevention rules.
**Immediately log any recognised mistake here** — including a clear "Prevention rule" so it is not repeated.
Search this file before starting any task in a module that has prior entries.

---

## Entry Template

Copy and fill in this template for every new entry:

```
### [YYYY-MM-DD] <Short title>

**Module/Area:** <e.g. users, api, migrations, settings>
**Symptom:** <What went wrong — observable behaviour or error message>
**Root cause:** <Why it happened>
**Fix:** <What was done to resolve it>
**Prevention rule:** <Concrete, actionable rule to prevent recurrence>
**Reference:** <File path(s), PR, issue, or commit if applicable>
```

---

## Entries

### [2026-03-17] Raw SQL constructed with string formatting creates SQL injection risk

**Module/Area:** database, ORM
**Symptom:** Queries built with f-strings or `%` string formatting pass user-controlled values directly into SQL strings.
**Root cause:** Developer bypassed the ORM for a complex query and used f-string interpolation for convenience.
**Fix:** Replace all string-formatted SQL with parameterised queries using `cursor.execute(sql, params)`.
**Prevention rule:** Never use f-strings, `.format()`, or `%` interpolation to construct SQL. Always pass user-supplied or variable values as parameters to `cursor.execute()`. Prefer the ORM; use raw SQL only with documented justification.
**Reference:** `config/settings/base.py` (ORM-first policy), `.github/copilot-instructions.md` §6.2

---

### [2026-03-17] Missing `permission_classes` on new DRF view allows unintended access

**Module/Area:** api, permissions
**Symptom:** A new API endpoint was accessible without authentication because it inherited only the global default, which was later changed.
**Root cause:** Developer omitted explicit `permission_classes` on the ViewSet, relying on the global `DEFAULT_PERMISSION_CLASSES`.
**Fix:** Add explicit `permission_classes = [IsAuthenticated]` (or stricter) to the ViewSet.
**Prevention rule:** Always declare `permission_classes` explicitly on every ViewSet and APIView, even if it matches the global default. Treat the global default as a last-resort fallback, not a design tool.
**Reference:** `config/settings/base.py` (`REST_FRAMEWORK.DEFAULT_PERMISSION_CLASSES`), `.github/copilot-instructions.md` §6.3

---

### [2026-03-17] Referencing `auth.User` directly breaks swappable user model

**Module/Area:** users, models, migrations
**Symptom:** `ValueError: ... is not a subclass of auth.User` or migration errors when the custom User model is loaded.
**Root cause:** Code imported `from django.contrib.auth.models import User` instead of using `get_user_model()`.
**Fix:** Replace all direct `User` imports with `from django.contrib.auth import get_user_model; User = get_user_model()` or use `settings.AUTH_USER_MODEL` in ForeignKey declarations.
**Prevention rule:** Never import `django.contrib.auth.models.User` anywhere in this project. Always use `get_user_model()` or `settings.AUTH_USER_MODEL`.
**Reference:** `salami/users/models.py`, `config/settings/base.py` (`AUTH_USER_MODEL`)

---

### [2026-03-17] Deleting or squashing migrations breaks deployed databases

**Module/Area:** migrations
**Symptom:** Deployment fails with `django.db.migrations.exceptions.InconsistentMigrationHistory` on target environments that had previously applied the deleted migration.
**Root cause:** Developer deleted or renumbered a migration file after it had been applied to a shared database.
**Fix:** Create a new corrective migration; never delete an already-applied migration.
**Prevention rule:** Never delete, rename, or squash existing migration files without first confirming that the migration has not been applied to any environment (local, staging, or production). If correction is needed, create a new migration.
**Reference:** `.github/copilot-instructions.md` §7 (Never do)

---

### [2026-03-17] N+1 queries caused by missing `select_related` on FK lookups

**Module/Area:** api, ORM, performance
**Symptom:** List API endpoints make one database query per object in the list (e.g. fetching the user for each salami record in a loop).
**Root cause:** QuerySet did not use `select_related()` for ForeignKey fields accessed in the serializer.
**Fix:** Add `select_related('user')` (or the relevant FK field) to the base QuerySet in the ViewSet's `get_queryset()` method.
**Prevention rule:** Whenever a serializer or view accesses a related object through a ForeignKey, verify that the QuerySet uses `select_related()` or `prefetch_related()` to avoid N+1 queries. Use Django Debug Toolbar (available locally) to confirm query counts.
**Reference:** `.github/copilot-instructions.md` §4.4

---

### [2026-03-17] `webpack-stats.json` missing causes cryptic startup error

**Module/Area:** frontend, webpack
**Symptom:** Django raises `WebpackError: Unable to load stats file` on page load when running without Docker.
**Root cause:** Webpack dev server was not started before the Django server, so `webpack-stats.json` was never generated.
**Fix:** Run `npm run dev` or `npm run build` before starting the Django server outside of Docker.
**Prevention rule:** When running Django locally without Docker, always start the Webpack dev server first (`npm run dev` in a separate terminal) or generate the stats file with `npm run build`. In Docker, the `node` service handles this automatically.
**Reference:** `config/settings/base.py` (`WEBPACK_LOADER`), `package.json`
