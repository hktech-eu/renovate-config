# PHP version sync

## The problem

A FrankenPHP image tag holds two versions: `dunglas/frankenphp:<frankenphp>-php<php>`,
for example `1.13.4-php8.5.12`. Not every pair is built. A new PHP patch ships only
on the current FrankenPHP release, so `1.12.7-php8.5.12` does not exist.

Two things went wrong with the default setup:

| Cause | Effect |
|---|---|
| Docker versioning reads `-php8.5.9` as a fixed suffix | Renovate never offers a new FrankenPHP tag |
| The `php` pin in `composer.json` was bumped on its own, and the Dockerfile suffix followed it | The PR named a tag that was never built, and the image build failed |

## The model

The image tag is the single source of truth for the PHP version.

1. The preset gives `dunglas/frankenphp` a `regex:` versioning. FrankenPHP
   major/minor/patch and the PHP patch can go up. The PHP minor (`8.5`) is the
   compatibility part, so a PHP minor upgrade stays a manual change.
2. The preset disables the composer `php` dependency, so Renovate never bumps it
   on its own.
3. The project syncs `composer.json` and `composer.lock` from the new tag with
   `postUpgradeTasks`.

Renovate only offers tags that exist on Docker Hub, so the PR always names an image
that can be pulled.

## Project setup

Extend the preset:

```json
"extends": ["github>hktech-eu/renovate-config//php-version-sync"]
```

Add this rule to `packageRules` in the project's `renovate.json`. Replace `<php-dir>`
with the directory that holds `composer.json`:

```json
{
  "description": "Sync the composer php pin and lock file from the FrankenPHP image tag",
  "matchDepNames": ["dunglas/frankenphp"],
  "postUpgradeTasks": {
    "commands": [
      "composer -d <php-dir> require --no-update --no-scripts --no-plugins php:{{{replace '^.*-php' '' newValue}}}",
      "composer -d <php-dir> update --lock --no-install --ignore-platform-reqs --no-scripts --no-plugins"
    ],
    "fileFilters": ["<php-dir>/composer.json", "<php-dir>/composer.lock"]
  }
}
```

Allowlist both commands in `.github/workflows/renovate.yml`:

```yaml
RENOVATE_ALLOWED_COMMANDS: '[
  "^composer -d <php-dir> require --no-update --no-scripts --no-plugins php:\\d+\\.\\d+\\.\\d+$",
  "^composer -d <php-dir> update --lock --no-install --ignore-platform-reqs --no-scripts --no-plugins$"
]'
```

| Project structure | `<php-dir>` |
|---|---|
| `composer.json` at repo root | `.` |
| PHP app under `app/` | `app` |
| PHP app under `api/` | `api` |

## What each command does

**`composer require --no-update php:<version>`**
Sets the exact `php` pin in `composer.json` to the PHP version from the new tag.
`{{{replace '^.*-php' '' newValue}}}` turns `1.13.4-php8.5.12` into `8.5.12`.

**`composer update --lock`**
Recomputes `content-hash` and updates `platform.php` in `composer.lock`. It does
not change any package. A plain `composer update` would update every package and
turn the PHP PR into a full dependency update.

## Gotchas

| Mistake | Effect |
|---|---|
| Rule not on `main` | Renovate reads config from the default branch and silently ignores it on any other branch |
| Command missing from `RENOVATE_ALLOWED_COMMANDS` | Renovate silently skips it |
| `fileFilters` path wrong | The file changes but is not committed |
| Any command fails | Renovate stops, and later commands do not run |
| A custom manager that bumps the PHP part of the tag on its own | The same "tag never built" failure comes back; remove it |
| `executionMode` defaults to repo root | Paths in commands and `fileFilters` are relative to the repo root |
