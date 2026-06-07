# PHP version sync

## The problem

When Renovate bumps the `php` version in `composer.json`, it can't install
the new PHP binary to regenerate `composer.lock` (the version isn't in
containerbase yet). Without extra configuration, only `composer.json` changes
— `composer.lock` and `Dockerfile` are left stale, and CI fails.

**What needs to happen in one PR:**

| File | What changes |
|---|---|
| `composer.json` | PHP version bumped by Renovate |
| `composer.lock` | `content-hash` recomputed, `platform.php` updated |
| `Dockerfile` | Base image PHP suffix updated (e.g. `-php8.4.1` → `-php8.4.2`) |

## The fix

Add this rule to `packageRules` in the project's `renovate.json`, adjusting
the paths to match where `composer.json`, `composer.lock`, and `Dockerfile`
live in that project:

```json
{
  "description": "Sync composer.lock and Dockerfile when the php version is bumped",
  "matchManagers": ["composer"],
  "matchDepNames": ["php"],
  "postUpgradeTasks": {
    "commands": [
      "composer -d <php-dir> update --no-install --ignore-platform-reqs --no-scripts",
      "sed -i \"s/\\\"php\\\": \\\"[0-9][^\\\"]*\\\"/\\\"php\\\": \\\"{{{newValue}}}\\\"/\" <php-dir>/composer.lock",
      "sed -i \"s/-php[0-9][0-9.]*/-php{{{newValue}}}/g\" <dockerfile>"
    ],
    "fileFilters": ["<php-dir>/composer.lock", "<dockerfile>"]
  }
}
```

Also add all three commands to `RENOVATE_ALLOWED_COMMANDS` in
`.github/workflows/renovate.yml`:

```yaml
RENOVATE_ALLOWED_COMMANDS: '[
  "^composer -d <php-dir> update --no-install --ignore-platform-reqs --no-scripts$",
  "^sed -i \"s/.+\" <php-dir>/composer\\.lock$",
  "^sed -i \"s/.+\" <dockerfile>$"
]'
```

## Path reference by project structure

| Structure | `<php-dir>` | `<dockerfile>` |
|---|---|---|
| `composer.json` at repo root | `.` (and drop `-d .`) | `Dockerfile` |
| PHP app under `api/` | `api` | `api/Dockerfile` |
| PHP app under `app/` | `app` | `app/Dockerfile` |

## What each command does

**`composer -d <php-dir> update --no-install --ignore-platform-reqs --no-scripts`**
Recomputes the `content-hash` in `composer.lock` to match the updated
`composer.json`. Without this, `composer install` in CI rejects the lock file
as stale. `--ignore-platform-reqs` allows running with the old PHP binary
even though `composer.json` now requires the new version.

**`sed -i "s/\"php\": \"...\"/.../\" <php-dir>/composer.lock`**
Patches the `platform.php` field in `composer.lock` to the new version.
Composer doesn't update this field automatically when only the `require.php`
constraint changes.

**`sed -i "s/-php[0-9][0-9.]*/-php{{{newValue}}}/g" <dockerfile>`**
Updates the PHP version suffix in the base image tag so that the Dockerfile,
`composer.json`, and `composer.lock` all reference the same PHP version in
one PR. Only applicable when using an image with an embedded PHP version in
its tag (e.g. FrankenPHP: `dunglas/frankenphp:1.x.x-php8.4.1`).

## Gotchas

| Mistake | Effect |
|---|---|
| Rule not on `main` | Renovate reads config from the default branch — silently ignored on any other branch |
| Wrong paths | Commands fail or modified files are discarded |
| `fileFilters` path wrong | File is modified but not committed |
| Any command in the sequence fails | Renovate aborts — subsequent commands never run |
| `$(jq ...)` instead of `{{{newValue}}}` | `jq` may not be in the Renovate container; use the built-in template variable |
| Command missing from `RENOVATE_ALLOWED_COMMANDS` | Renovate silently skips it |
| `executionMode` defaults to repo root | All paths in commands and `fileFilters` are relative to the repo root, not the package file directory |
